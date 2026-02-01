# Chat

User-facing chat for querying repository documentation and RAG context.

## Scope

- Uses internal Docrunch LLM profiles (Manager, Librarian, QA, Security, UI/UX).
- Not the same as external CLI coding agents.
- Default search scope is docs; reports are opt-in.

## Data Flow

```
UI -> Chat API -> QueryEngine (docs + RAG) -> LLM -> Response -> Postgres
```

## Data Model

```python
@dataclass
class ChatSession:
    id: str
    title: str
    created_at: datetime
    updated_at: datetime

@dataclass
class ChatMessage:
    id: str
    session_id: str
    role: str  # user | assistant | system
    content: str
    citations: list[str]
    created_at: datetime
```

## REST Endpoints

| Method | Endpoint                           | Description                  |
| ------ | ---------------------------------- | ---------------------------- |
| GET    | `/api/chat/sessions`               | List chat sessions           |
| POST   | `/api/chat/sessions`               | Create chat session          |
| GET    | `/api/chat/sessions/{id}/messages` | List session messages        |
| POST   | `/api/chat/sessions/{id}/messages` | Send message + get response  |

## UX Expectations

- Conversation list and search
- Citation chips to source docs
- Scope selector (docs, reports, all)
- Streaming responses (optional)

## Implementation Plan

1. Add Postgres tables for chat sessions/messages
2. Implement ChatService with RAG retrieval + LLM call
3. Add REST endpoints (streaming optional)
4. Add dashboard chat UI
5. Log and audit chat usage
