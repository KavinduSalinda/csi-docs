# Pipeline notes

[← Documentation home](README.md)

Operational dependencies, coverage characteristics, and remaining product gaps. This is **not** a list of broken pipeline steps — most items below are expected behavior or external prerequisites.

---

## External dependencies

| Dependency | Used by | Notes |
|------------|---------|-------|
| `OPENROUTER_API_KEY` | DAG 02 sponsor AI, DAG 05 facility cleanup, DAG 08 Postgres enrichment | Required for OpenRouter populate tasks |
| `GOOGLE_APPLICATION_CREDENTIALS` / GCS / BQ | DAGs 01–06 | Service account with BigQuery + Storage access |
| `DB_*` | DAGs 07–09 | Pipeline Postgres (separate from Airflow metadata unless `AIRFLOW_DB_NAME` set) |
| `NEO4J_*` | DAG 10 | Graph load credentials |
| Network egress | DAG 01 | AACT download, OpenAlex S3, WHO ICD-10 zip, Zenodo ROR |

Optional OpenRouter tuning: `OPENROUTER_ENRICHMENT_MODEL`, `OPENROUTER_SPONSOR_INSIGHTS_MODEL`, `SPONSOR_AI_INSIGHTS_*`.

---

## Operational characteristics

| Topic | Detail |
|-------|--------|
| OpenAlex **works** ingest | Large (~250GB compressed); first DAG 01 run may take many hours |
| OpenAlex **authors** ingest | Separate S3 snapshot in DAG 01 (`open_alex_bronze.authors`); required for `pi_master.h_index` / `i10_index` |
| ICD-10 hierarchy | One-time WHO meta load in DAG 02; skips reload when table exists (`ICD10_SKIP_IF_EXISTS`) |
| Weekly chain | DAG 01 scheduled; DAGs 02–10 trigger on prior success only |
| DAG 05 / 08 incremental | Only unscored/unrefined facilities or unmapped countries/conditions processed by default |
| Export manifest | DAG 06 exports a fixed table list — not every BQ table |
| Graph schema | DAG 10 loads a defined Neo4j model only |
| Airflow host | Linux/WSL/VPS — not native Windows |

---

## Coverage and data quality

| Area | Behavior |
|------|----------|
| **`pi_master` OpenAlex match** | Name-based join to works authorships; `h_index` / `i10_index` from `open_alex_bronze.authors.summary_stats` when `openalex_id` matches |
| **Unmatched investigators** | NULL OpenAlex fields when trial name ≠ OpenAlex display name |
| **`sponsor_company_info`** | ROR exact-name match for INDUSTRY sponsors — HQ city/country only; website/size/founded year not populated |
| **`sponsor_master` ROR columns** | NULL in silver; full sponsor–ROR mapping via gold `facility_ror_mapping` still pending |
| **`icd10_hierarchy_with_mesh`** | MeSH rows only where ICD-10 labels match AACT `browse_conditions_raw` terms |
| **Gold PI publications** | From OpenAlex works snapshot — stale until next DAG 01 run |

---

## Remaining product gaps

Work not yet implemented (honest backlog):

- Sponsor `ror_id` / OpenAlex fields on `sponsor_master` (gold ROR mapping port)
- Richer `sponsor_company_info` (website, size, founded year beyond ROR HQ)
- Fuzzy / LLM investigator–OpenAlex disambiguation (beyond name match)
- `i10_index` in canonical `investigator` and Postgres export (silver `pi_master` has it; downstream does not yet)

---

## Related documentation

- [Data layers](data-layers.md)
- [DAG reference](dags/README.md)
- Per-DAG notes sections link here from individual DAG pages

[← Back to documentation home](README.md)
