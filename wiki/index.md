# Wiki Index

## Platform

- [Sites Intelligence Platform](sites-intelligence.md) — Overview of the platform, capabilities, and repository layout.

## Technical Architecture

- [Architecture](architecture.md) — Backend stack (Django, PostgreSQL, Neo4j, Spanner, OpenAI) and API surfaces.
- [Breakdown Pipeline](breakdown-pipeline.md) — Async workflow for generating feasibility reports.

## Study Creation

- [Study Creation](study-creation.md) — Manual and assisted creation modes, API, data model, edge cases.
- [Study Creation AI Modes](study-creation-ai-modes.md) — Assisted extraction pipeline, hybrid assistant, AI extraction constraints.
- [File Uploads](file-uploads.md) — Two-step upload architecture, file type support, text extraction.
- [File Summarization](file-summarization.md) — Async background summarization of attached documents.

## Breakdown Scoring

- [Breakdown Scoring](breakdown-scoring.md) — Weighted A–D category scoring system (0–100 match score), API, generation order.
- [Breakdown Category A](breakdown-category-a.md) — A1 Therapeutic Area, A2 Phase Experience, A3 Capabilities, A4 Competing Trials.
- [Breakdown Category B](breakdown-category-b.md) — B1 Enrollment Performance; B2/B3 planned.
- [Breakdown Category C](breakdown-category-c.md) — C1 PI Experience, C2 PI Publication Quality; shared PI pipeline.
- [Breakdown Category D](breakdown-category-d.md) — D1 Country Regulatory Environment, D2 Institution Type.

## AI Features

- [AI Features](ai-features.md) — Inventory, shared infrastructure, environment variables, performance and accuracy expectations.
- [Entity Chats](entity-chats.md) — Site chat, sponsor chat, sponsor insights chat, helpful feedback.
- [PI and Breakdown AI](pi-and-breakdown-ai.md) — PI relevance scoring, LLM insights, section summaries.

## Site Discovery

- [Natural Search](natural-search.md) — Natural-language search for sites and PIs; Cypher generation, sidebar filters, limitations.
- [Flexibility Modes and AI Suggested Sites](flexibility-modes.md) — Graph-based site matching, strict/balanced/exploratory modes, sidebar overrides.

## Administration

- [Override Panel](override-panel.md) — Admin data overrides for sites, PIs, and study history; deactivation, audit trail, impact gaps.
