# Pipeline overview

[← Documentation home](README.md)

This page describes the **end-to-end Sites Intelligence pipeline**: where data comes from, how it is transformed, and where it lands.

---

## Purpose

The pipeline supports **clinical site intelligence** by connecting:

- **Trial sites (facilities)** and their equipment, geography, and performance signals  
- **Studies** (trials), phases, conditions, and MeSH terms  
- **Investigators** and their publications  
- **Sponsors** and competitive landscape information  

The final outputs power **PostgreSQL-backed applications** and a **Neo4j knowledge graph** for relationship queries.

---

## Systems involved

```mermaid
flowchart TB
    subgraph sources [Data sources]
        AACT[AACT clinical trials]
        OA[OpenAlex publications]
        GCS_IN[Files in Cloud Storage]
    end

    subgraph gcp [Google Cloud]
        BQ[BiqQuery<br/>analytics layers]
        GCS[Cloud Storage<br/>CSV exchange]
    end

    subgraph apps [Downstream systems]
        PG[PostgreSQL<br/>application DB]
        N4J[Neo4j<br/>knowledge graph]
    end

    AF[Apache Airflow<br/>orchestration]

    AACT --> GCS_IN
    GCS_IN --> BQ
    OA --> BQ
    AF --> BQ
    AF --> GCS
    BQ --> GCS
    GCS --> PG
    PG --> GCS
    GCS --> N4J
```

| System | Role |
|--------|------|
| **Apache Airflow** | Schedules and chains pipeline steps; retries on failure |
| **BigQuery** | Stores bronze → silver → gold → canonical layers |
| **Cloud Storage** | Holds AACT flat files and exported CSVs between steps |
| **PostgreSQL** | Operational relational database for the product |
| **Neo4j** | Graph database for connected queries (sites, studies, people, sponsors) |

---

## Ten pipeline steps (DAGs 01–10)

The pipeline is split into **ten Airflow DAGs** that run **in order**. Each DAG must finish successfully before the next starts.

```mermaid
flowchart LR
    subgraph phase1 [Prepare data in BigQuery]
        direction TB
        A1["01 Bronze<br/><i>raw load</i>"]
        A2["02 Silver<br/><i>clean & join</i>"]
        A3["03 Gold<br/><i>enrich</i>"]
        A4["04 Canonical<br/><i>unified model</i>"]
        A5["05 Facility cleanup<br/><i>AI names</i>"]
    end
    subgraph phase2 [Deliver to products]
        direction TB
        B6["06 Export CSVs"]
        B7["07 Load Postgres"]
        B8["08 Postgres enrichment"]
        B9["09 Export for graph"]
        B10["10 Load Neo4j"]
    end
    A1 --> A2 --> A3 --> A4 --> A5 --> B6 --> B7 --> B8 --> B9 --> B10
```

| Phase | DAGs | What happens |
|-------|------|----------------|
| **Ingest** | 01 | Raw trial and publication data loaded into BigQuery “bronze” tables |
| **Transform** | 02–03 | Data cleaned (silver) and enriched (gold) |
| **Model** | 04 | Canonical `scl_staging` tables — the product-facing shape in BigQuery |
| **Quality** | 05 | Facility names copied to a cleaned dataset, scored, and refined with AI |
| **Export** | 06 | Canonical tables written as CSV files in Cloud Storage |
| **Load** | 07 | CSVs loaded into PostgreSQL |
| **Enrich** | 08 | OpenRouter populate + SQL updates link sites to regions and therapeutic areas |
| **Graph prep** | 09 | Selected Postgres tables exported again for the graph loader |
| **Graph** | 10 | Nodes and relationships loaded into Neo4j |

Step-by-step detail: [DAG reference](dags/README.md)

---

## Scheduling

| DAG | Typical schedule |
|-----|------------------|
| **DAG 01** | Weekly (`@weekly`) — pipeline entry point |
| **DAGs 02–10** | Triggered automatically when the previous DAG succeeds |

A full run duration depends on data volume, API calls (facility cleanup), and infrastructure. Bronze and silver steps are the heaviest BigQuery work; graph load (DAG 10) depends on export size.

---

## Data layer summary

BigQuery uses a **medallion** pattern:

```mermaid
flowchart LR
    BR[Bronze<br/>raw copies]
    SV[Silver<br/>cleaned]
    GD[Gold<br/>enriched]
    CN[Canonical<br/>scl_staging]
    CL[Cleaned<br/>scl_staging_cleaned]

    BR --> SV --> GD --> CN --> CL
```

Full dataset list: [Data layers](data-layers.md)

---

## What to read next

- [Data layers](data-layers.md) — table locations and naming  
- [Pipeline notes](pipeline-notes.md) — dependencies, AI keys, and coverage characteristics  
- [DAG 01 — Bronze](dags/01-bronze-layer.md) — start of the step-by-step guide  

[← Back to documentation home](README.md)
