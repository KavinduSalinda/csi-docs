---
title: AI Features
created: 2026-06-21
updated: 2026-06-21
tags: [ai, llm, openai, openrouter, overview]
sources: ["[[raw/ai-features/README.md]]", "[[raw/ai-features/limitations.md]]"]
---

# AI Features

Central reference for LLM-powered capabilities in Sites Intelligence. These features use OpenAI and/or OpenRouter to generate text, extract structured data, score entities, or answer questions grounded in platform data.

## Details

### Feature Inventory

| Feature | Type | Endpoint / trigger | Model |
|---------|------|-------------------|-------|
| Site chat | Q&A | `POST /api/v1/sites/{id}/chat/` | OpenAI `gpt-5.2` |
| Sponsor chat | Q&A | `POST /api/v1/sponsors/{id}/chat/` | OpenAI `gpt-5.2` |
| Sponsor insights chat | Q&A | `POST /api/v1/sponsors/{id}/insights/chat/` | OpenAI `gpt-5.2` |
| Chat helpful feedback | UX metadata | `PATCH /api/v1/users/chat/{conv_id}/message/{msg_id}/` | N/A |
| Assisted study creation | Structured extraction | `POST /api/v1/studies/?mode=assisted` | OpenRouter (default) |
| Study assistant | Chat + field updates + clarifications | `POST /api/v1/studies/{id}/assistant/` | OpenRouter (default) |
| Document summarization | Background summary | On `POST /studies/{id}/documents/` attach | OpenRouter (default) |
| PI relevance scoring | Scoring | Breakdown generation (C1) | OpenAI `gpt-5.2` |
| Breakdown LLM insights | Narrative | Breakdown generation | OpenAI `gpt-4o-mini` |
| Breakdown section summaries | Summarization | Breakdown generation | OpenRouter `gpt-4o-mini` |

**Important scoping notes:**
- **Study AI** has a single endpoint — `POST /api/v1/studies/{id}/assistant/`. There is no separate study Q&A chat.
- **PI Q&A chat** does not exist. PI AI is **relevance scoring** during breakdown only.
- **Natural language search** and **AI suggested sites** use OpenAI for **query generation** against graph databases — they are graph features, not LLM recommendation features. Documented in [[Natural Search]] and [[Overrides and Flexibility]].

### Shared Infrastructure

| Module | Role |
|--------|------|
| `csi/openai_chat.py` | `get_openai_client()`, `get_chat_response()` — chats and PI scoring |
| `csi/openrouter_service.py` | `call_ai()` — extraction, assistant, summarization, breakdown summaries |
| `users.models.AIChatConversation` | Shared conversation store for entity chats |
| `studies.models.StudyAssistantSession` | Study creation / workspace assistant sessions |

**Temperature defaults:** 0.3 for chats; 0.1 for extraction; 0.2 for summarization; 0.7 for LLM insights; 0.3 for section summaries.

### Environment Variables

| Variable | Required for |
|----------|--------------|
| `OPENAI_API_KEY` | Site/sponsor chats, PI scoring, breakdown LLM insights |
| `OPENROUTER_API_KEY` | Protocol extraction, study assistant, document summarization, breakdown section summaries |

Missing `OPENAI_API_KEY` → site/sponsor chat endpoints return **503**. OpenRouter failures return structured errors or fallbacks depending on the feature.

Test flag `USE_OPENAI_SUMMARY_FOR_TESTING = True` routes OpenRouter features through OpenAI instead.

### Platform-Wide AI Limitations

| Limitation | Detail |
|------------|--------|
| No real-time web access | All features use only data provided in the prompt context |
| Hallucination risk | System prompts forbid invention; models may still infer beyond source data |
| API key dependency | Missing keys disable respective features |
| No response caching | Chats and scoring repeat full model calls each request |
| English-primary | Prompts and extraction optimized for English protocol text |
| Cost scales with usage | Large contexts, many PIs, and long chats increase token spend |

### Performance Expectations

| Feature | Typical latency | Notes |
|---------|-----------------|-------|
| Entity chat | 2–15+ seconds | Depends on context size |
| Assistant turn | 5–30+ seconds | Large doc context |
| Protocol extraction | 10–60+ seconds | Synchronous at create |
| Document summary | 5–30 seconds | Background; not user-blocking |
| Full breakdown | Minutes | Dominated by PI count × scoring calls |

### Accuracy Expectations

| Feature | Trust level | Recommendation |
|---------|-------------|----------------|
| Entity chats | Informational | Verify critical facts against source records |
| Protocol extraction | Draft bootstrap | Human review required |
| Assistant field updates | Semi-automated | Review applied changes in study workspace |
| PI relevance scores | Ranking aid | Use holistically, not score alone |
| LLM insights | Narrative summary | Supplement to numeric breakdown, not replacement |
| Section summaries | Executive preview | Fallback templates are generic |

## Related

- [[Entity Chats]]
- [[Study Creation AI Modes]]
- [[File Summarization]]
- [[PI and Breakdown AI]]
- [[Breakdown Category C]]
- [[Natural Search]]
- [[Overrides and Flexibility]]

## Open Questions
