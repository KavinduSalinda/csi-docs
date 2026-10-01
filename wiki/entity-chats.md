---
title: Entity Chats
created: 2026-06-21
updated: 2026-06-21
tags: [ai, chat, site-chat, sponsor-chat, openai, conversation]
sources: ["[[raw/ai-features/entity-chats.md]]", "[[raw/ai-features/limitations.md]]"]
---

# Entity Chats

Read-only conversational Q&A for **sites** and **sponsors**. Each chat grounds answers in serialized entity context and optional conversation history. Answers are never used to modify entity records.

## Details

### Shared Request/Response Pattern

All entity chat endpoints share the same shape:

**Request:**
```json
{
  "question": "What therapeutic areas does this site specialize in?",
  "chat_id": 42
}
```
`chat_id` is optional — omit to start a new conversation.

**Response:**
```json
{
  "data": {
    "answer": "The site specializes in Oncology and Cardiology.",
    "chat_id": 42,
    "answer_id": 108
  },
  "code": 200,
  "message": "Chat response generated"
}
```

### Shared Flow

1. Serialize entity context → append to system prompt.
2. Resolve or create `AIChatConversation` for the authenticated user.
3. Load prior messages as `chat_history` (full history replayed each turn — no summarization trim).
4. Call `get_chat_response(system_prompt, user_message, chat_history, content)`.
5. Persist user + assistant messages; return `answer`, `chat_id`, `answer_id`.

Model: **OpenAI `gpt-5.2`**, temperature **0.3**. System prompts enforce **grounded answers only** — no external knowledge, no invented fields.

### Error Handling

| Condition | HTTP |
|-----------|------|
| `OPENAI_API_KEY` not configured | 503 |
| OpenAI API failure | 500 |
| Invalid serializer input | 400 |

Chats do **not** fail-soft — missing AI configuration surfaces as 503. No graceful degraded answer.

---

### Site Chat

**Endpoint:** `POST /api/v1/sites/{id}/chat/`  
**Code:** `sites/views.py` → `SiteChatAPIView`

**Context loaded:** `SiteOverviewSerializer` output for the site, including override context when active. Covers site attributes, performance metrics, and characteristics present in the serializer.

---

### Sponsor Chat

**Endpoint:** `POST /api/v1/sponsors/{id}/chat/`  
**Code:** `sponsors/views.py` → `SponsorChatAPIView`

**Context loaded (composite JSON):**

| Section | Source |
|---------|--------|
| `sponsor.details` | `SponsorDetailSerializer` |
| `history` | Sponsor-linked `StudyHistory` records |
| `collaboration_summary` | Total studies, completed count, success rate |
| `events` | `EventAndActivity` |
| `highlights` | `Highlights` |
| `sponsor_intelligence` | `SponsorIntelligence` |
| `sponsor_lesson_learned` | `LessonLearned` |
| Related studies | Study metadata linked to sponsor |

---

### Sponsor Insights Chat

**Endpoint:** `POST /api/v1/sponsors/{id}/insights/chat/`  
**Code:** `sponsors/views.py` → `SponsorInsightsChatAPIView`

Narrower scope — focuses on intelligence and lessons learned only.

**Context loaded:**
```json
{
  "sponsor": { "id": 7, "name": "..." },
  "sponsor_intelligence": [ ... ],
  "sponsor_lesson_learned": [ ... ]
}
```

| Chat | Best for |
|------|----------|
| Sponsor chat | Full sponsor profile, history, events, collaboration metrics |
| Insights chat | Intelligence records and lessons learned only |

---

### Helpful Feedback

**Endpoint:** `PATCH /api/v1/users/chat/{conversation_id}/message/{message_id}/`

Allows users to mark an assistant reply as helpful (`is_helpful: true/false`). Used for UX analytics — does not affect future AI responses. Requires auth; conversation must belong to requesting user.

---

### Limitations

| Constraint | Detail |
|------------|--------|
| Read-only | Site and sponsor chats never modify entity records |
| Context completeness | Answers limited to serialized fields; missing DB data → "not available" |
| History unbounded | Full conversation replayed each turn — very long chats increase latency and cost |
| No study entity chat | Study AI is only `POST /api/v1/studies/{id}/assistant/` |
| No PI chat endpoint | PI Q&A is not implemented; PI AI = relevance scoring during breakdown |

## Related

- [[AI Features]]
- [[PI and Breakdown AI]]
- [[Study Creation AI Modes]]

## Open Questions
