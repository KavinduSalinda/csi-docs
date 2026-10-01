---
title: Sites Intelligence Platform
created: 2026-06-21
updated: 2026-06-21
tags: [platform, overview, clinical-trials, site-selection]
sources: ["[[raw/README.md]]"]
---

# Sites Intelligence Platform

Sites Intelligence is a clinical trial site selection platform that helps sponsors and study teams evaluate how well a research site fits a specific study, discover candidate sites with AI-assisted search, and make data-driven feasibility decisions before site activation.

## Details

### What the Platform Does

| Capability | Description |
|------------|-------------|
| **Study creation** | Define protocol requirements: phase, therapeutic area, MeSH conditions, capabilities, enrollment targets, and documents. |
| **Site discovery** | Search and filter sites using structured filters, natural-language queries, and AI-suggested site matching. |
| **Study–site breakdown** | Generate a weighted, multi-section feasibility report for any study–site pair with an overall match score. |
| **PI evaluation** | Rank principal investigators at a site using trial history, publications, and AI relevance scoring. |
| **Overrides & flexibility** | Apply site/PI overrides and match-flexibility modes when real-world constraints differ from strict protocol fit. |

### Main Features

- **Weighted breakdown scoring** — Four category groups (A–D) with eleven scored sub-sections produce a 0–100 match score.
- **Background report generation** — Breakdown jobs run asynchronously with progress tracking and versioning.
- **LLM narrative insights** — Strengths, risks, and recommended next steps generated from the full breakdown context.
- **MeSH hierarchy drill-down** — A1 exposes therapeutic-area trial experience as a navigable MeSH tree.
- **Competing trials analysis** — A4 quantifies enrollment competition and site capacity load.
- **ETL reference data** — Countries, institution types, MeSH hierarchy, and site master data refreshed from CSI ETL pipelines.

### Repository Layout

| Path | Role |
|------|------|
| `studies/` | Study models, breakdown generators, study APIs |
| `sites/` | Site models, search/matching services, overrides |
| `principal_investigators/` | PI and publication models |
| `csi/` | Shared services — OpenAI, Neo4j, Spanner, ETL |
| `docs/` | Product and technical documentation |
| `dev-docs/` | Internal engineering notes and migration guides |
| `/users` | User authentication, profile management, role assignment, and access control |

## Related

- [[Architecture]]
- [[Breakdown Pipeline]]
- [[Breakdown Scoring]]
- [[Study Creation]]
- [[Site Discovery]]
- [[PI Evaluation]]
- [[Overrides and Flexibility]]
- [[Natural Search]]

## Open Questions
