# Session Memory

Persistent conversation history and context management for AI agents across sessions.

## Overview

Session Memory solves the "context loss" problem where AI agents forget previous conversations and decisions. It stores conversation history, files discussed, and decisions made for retrieval in future sessions.

Current session:
- Agent: claude
- Task: task-123
- Messages: 45
- Files discussed: [auth.py, users.py]
- Decisions: 3

Previous sessions (task-123):
- Session 1: Gemini - Research phase
- Session 2: Claude - Implementation
- Session 3: Claude - Bug fixes

## Data Model

```python
@dataclass
class Session:
    id: str
    task_id: str | None
    agent: str  # claude, gemini, codex
    started_at: datetime
    ended_at: datetime | None

    # Conversation
    messages: list[Message]

    # Context
    files_discussed: list[str]
    components_referenced: list[str]

    # Outcomes
    decisions_made: list[Decision]
    findings: list[Finding]
    code_changes: list[str]

@dataclass
class Message:
    role: str  # user, assistant, system
    content: str
    timestamp: datetime
    tokens: int
    tool_calls: list[ToolCall] | None
```

## Features

### 1. Automatic Context Injection

When an agent claims a task, inject relevant session history:

```python
def get_task_context(task_id: str) -> TaskContext:
    # Get previous sessions for this task
    sessions = get_sessions_for_task(task_id)

    return TaskContext(
        previous_sessions=summarize_sessions(sessions),
        key_decisions=extract_decisions(sessions),
        files_to_review=extract_discussed_files(sessions),
        warnings=extract_blockers(sessions)
    )
```

### 2. Session Summarization

Automatically summarize long sessions to fit context windows:

```python
def summarize_session(session: Session) -> str:
    """Compress session to key points."""
    return llm.summarize(f"""
        Summarize this coding session:
        - What was discussed
        - What decisions were made
        - What code was changed
        - Any open issues
    """)
```

### 3. Cross-Session Search

Search across all sessions for relevant context:

```python
@server.call_tool()
async def search_sessions(query: str, task_id: str = None) -> list[SessionResult]:
    """Search session history for relevant context."""
```

## MCP Tools

| Tool                   | Description                            |
| ---------------------- | -------------------------------------- |
| `get_session_history`  | Get previous sessions for current task |
| `search_sessions`      | Search all sessions by query           |
| `log_decision`         | Log a decision to current session      |
| `log_file_reference`   | Mark file as discussed                 |
| `get_related_sessions` | Find sessions about similar topics     |

## Storage

Sessions stored in Postgres:

```sql
CREATE TABLE sessions (
    id TEXT PRIMARY KEY,
    task_id TEXT,
    agent TEXT,
    started_at TIMESTAMP,
    ended_at TIMESTAMP,
    summary TEXT,
    tokens_used INTEGER
);

CREATE TABLE session_messages (
    id TEXT PRIMARY KEY,
    session_id TEXT,
    role TEXT,
    content TEXT,
    timestamp TIMESTAMP,
    tokens INTEGER,
    FOREIGN KEY (session_id) REFERENCES sessions(id)
);

CREATE TABLE session_files (
    session_id TEXT,
    file_path TEXT,
    action TEXT,  -- discussed, modified, created
    FOREIGN KEY (session_id) REFERENCES sessions(id)
);
```

## Configuration

```yaml
# .docrunch/config.yaml
session:
  enabled: true
  max_messages_stored: 1000
  summarize_after: 50 # messages
  retention_days: 90

  context_injection:
    enabled: true
    max_sessions: 5
    include_decisions: true
    include_files: true
```
