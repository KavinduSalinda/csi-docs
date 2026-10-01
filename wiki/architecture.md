---
title: Architecture
created: 2026-06-21
updated: 2026-06-21
tags: [architecture, backend, django, neo4j, spanner, postgresql]
sources: ["[[raw/README.md]]"]
---

# Architecture

The Sites Intelligence backend (`sites-intelligence-be`) is a Django REST API that combines structured trial data, graph/relational queries, ETL-fed reference catalogs, and LLM-assisted analysis.

## Details

### Tech Stack

- **Django REST Framework** — primary API layer
- **PostgreSQL / Django ORM** — transactional data (studies, sites, users, breakdown results)
- **Neo4j** — graph queries for site/PI relationships and matching (feature-flagged)
- **Google Spanner** — search workloads (feature-flagged per endpoint)
- **OpenAI** — LLM integration for PI scoring, narrative insights, and study creation AI

Both Neo4j and Spanner are **dual data backends** — endpoints can be toggled between them via feature flags.

### Key API Surfaces

| Area | Base path | Purpose |
|------|-----------|---------|
| Studies | `/api/v1/studies/` | Study CRUD, breakdown, history |
| Sites | `/api/v1/sites/` | Site listing, search, overrides |
| Breakdown | `/api/v1/studies/<study_id>/sites/<site_id>/breakdown/` | Generate and retrieve feasibility reports |
| Breakdown jobs | `/api/v1/studies/breakdown-jobs/<job_id>/` | Poll async job status |
| Users | `/api/v1/users` | User identity, authentication, roles, and access control |

### ETL Integration

Reference data (countries, institution types, MeSH hierarchy, site master data) is refreshed from CSI ETL pipelines into the local database, making it available for structured filtering and matching.

## Related

- [[Sites Intelligence Platform]]
- [[Breakdown Pipeline]]
- [[Breakdown Scoring]]
- [[Natural Search]]

## Open Questions
