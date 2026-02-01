# Interface Contracts

> **Note:** This document is the **authoritative source** for all Pydantic model definitions. Other component docs may reference these models but should link here for the canonical schema.

## Core <-> Storage Interface

### Scanner Output

```python
@dataclass
class ScanResult:
    """Output from Scanner, consumed by Storage."""
    root_path: Path
    file_tree: FileNode
    total_files: int
    total_directories: int
    scan_duration_ms: int
    errors: list[ScanError]

@dataclass
class FileNode:
    path: Path
    name: str
    type: Literal["file", "directory"]
    language: str | None
    size_bytes: int
    modified_at: datetime
    content_hash: str
    children: list[FileNode]  # If directory
```

### Parser Output

```python
@dataclass
class Import:
    """Represents a single import target."""
    module: str
    name: str | None  # None for `import module`; "*" for wildcard import
    alias: str | None
    start_line: int
    end_line: int

@dataclass
class Export:
    """Represents a single export target."""
    name: str | None  # None for `export *`
    alias: str | None
    is_default: bool
    source: str | None  # Module for re-exports
    start_line: int
    end_line: int

@dataclass
class Function:
    """Represents a function definition."""
    name: str
    start_line: int
    end_line: int
    signature: str | None
    docstring: str | None

@dataclass
class Class:
    """Represents a class definition."""
    name: str
    start_line: int
    end_line: int
    bases: list[str]
    docstring: str | None

@dataclass
class Schema:
    """Represents a schema definition (SQL/ORM)."""
    name: str
    kind: str | None  # table, view, model, etc.
    start_line: int | None
    end_line: int | None

@dataclass
class ParsedFile:
    """Output from Parser, consumed by Storage."""
    file_path: Path
    language: str
    success: bool
    error: str | None

    imports: list[Import]
    exports: list[Export]
    functions: list[Function]
    classes: list[Class]
    components: list[Component]
    schemas: list[Schema]
```

---

## MCP Report Contracts (MCP -> Storage)

```python
@dataclass
class ProgressUpdate:
    task_id: str
    percentage: int
    status: str
    reported_by: str
    reported_at: datetime

@dataclass
class CompletionReport:
    task_id: str
    summary: str
    changes: list[str]
    tests_passed: bool
    notes: str | None
    findings: list[Finding]
    decisions: list[Decision]
    reported_by: str
    reported_at: datetime

@dataclass
class Finding:
    id: str
    task_id: str
    type: Literal["bug", "improvement", "question", "blocker"]
    description: str
    files: list[str] | None
    created_by: str
    created_at: datetime

@dataclass
class Decision:
    id: str
    title: str
    context: str
    decision: str
    rationale: str
    created_at: datetime
    created_by: str

@dataclass
class BlockerReport:
    task_id: str
    description: str
    needs: str
    reported_by: str
    reported_at: datetime
```

Report persistence (Postgres):

-   task_history stores claim/progress/completion events (action + details)
-   completion_reports stores final completion reports per task
-   findings, decisions, blockers store structured report records

### Required Fields (Report Schema)

-   claim_task: task_id, reported_by, reported_at
-   update_progress: task_id, percentage, status, reported_by, reported_at
-   submit_completion: task_id, summary, changes, tests_passed, reported_by, reported_at
-   log_finding: task_id, type, description, created_by, created_at
-   record_decision: title, context, decision, rationale, created_by, created_at
-   report_blocker: task_id, description, needs, reported_by, reported_at

Optional fields: notes, findings, decisions, files, status, resolved_at.
Server assigns IDs when not provided by client.

### Markdown Publishing

-   Completion reports publish to Markdown `tasks/{task_id}.md` on completion (explicit publish is planned)
-   Findings and decisions can be summarized in task reports; canonical data remains in Postgres

---

## Storage <-> MCP Interface

### Query Interface

```python
class StorageQuery(Protocol):
    """Interface that Storage exposes to MCP."""

    async def search(
        self,
        query: str,
        limit: int = 10,
        scope: Literal["docs", "reports", "all"] = "docs",
    ) -> list[SearchResult]

    async def get_component(
        self,
        name: str
    ) -> Component | None

    async def get_relations(
        self,
        path: str
    ) -> list[Relationship]

    async def get_file_metadata(
        self,
        path: str
    ) -> FileMetadata | None

@dataclass
class SearchResult:
    id: str
    type: str  # function, class, module, report, decision, finding, task
    title: str
    content: str
    score: float
    file_path: str | None  # None for non-file results (decisions, findings, reports, tasks)
    line_number: int | None  # None when file_path is None or for file-level results
```

> **Note:** `file_path` and `line_number` are `None` for entity types that don't correspond to specific source files, such as decisions, findings, reports, and tasks.

---

## MCP <-> Dashboard Interface

### REST API Endpoints

| Method | Endpoint                                           | Description                         |
| ------ | -------------------------------------------------- | ----------------------------------- | ------- | ---- |
| GET    | `/api/health`                                      | Health check                        |
| GET    | `/api/search`                                      | Search docs and reports (scope=docs | reports | all) |
| GET    | `/api/modules`                                     | List all modules                    |
| GET    | `/api/modules/{name}`                              | Get module details                  |
| GET    | `/api/functions/{name}`                            | Get function details                |
| GET    | `/api/tasks`                                       | List tasks                          |
| GET    | `/api/tasks/{id}`                                  | Get task details                    |
| PATCH  | `/api/tasks/{id}`                                  | Update task                         |
| GET    | `/api/graph`                                       | Get knowledge graph                 |
| GET    | `/api/llm/providers`                               | List LLM providers                  |
| POST   | `/api/llm/providers`                               | Create LLM provider                 |
| PATCH  | `/api/llm/providers/{id}`                          | Update LLM provider                 |
| GET    | `/api/llm/providers/{id}/models`                   | List provider models                |
| POST   | `/api/llm/providers/{id}/models`                   | Add provider model                  |
| GET    | `/api/llm/cli-models`                              | List all CLI model sets             |
| GET    | `/api/llm/cli-models/{cli_type}`                   | Get CLI models for a type           |
| PUT    | `/api/llm/cli-models/{cli_type}`                   | Replace CLI models for a type       |
| POST   | `/api/llm/cli-models/{cli_type}/models`            | Add CLI model                       |
| DELETE | `/api/llm/cli-models/{cli_type}/models/{model_id}` | Delete CLI model                    |
| GET    | `/api/llm/profiles`                                | List LLM profiles                   |
| GET    | `/api/llm/profiles/{role}`                         | Get LLM profile                     |
| PATCH  | `/api/llm/profiles/{role}`                         | Update LLM profile                  |
| GET    | `/api/chat/sessions`                               | List chat sessions                  |
| POST   | `/api/chat/sessions`                               | Create chat session                 |
| GET    | `/api/chat/sessions/{id}/messages`                 | List messages                       |
| POST   | `/api/chat/sessions/{id}/messages`                 | Send message + response             |

### Response Format

```typescript
interface ApiResponse<T> {
    success: boolean;
    data: T | null;
    error: ApiError | null;
}

interface ApiError {
    code: string;
    message: string;
    details?: Record<string, unknown>;
}
```

---

## LLM Settings Contracts (Dashboard -> Storage)

```python
@dataclass
class LLMProvider:
    id: str
    name: str
    provider_type: Literal["openai", "anthropic", "xai", "google", "ollama", "cli_bridge"]
    is_enabled: bool
    config: dict  # api_key, base_url, cli_type, proxy_url, timeout, etc.
    created_at: datetime
    updated_at: datetime

@dataclass
class LLMProfile:
    id: str
    role: Literal["manager", "librarian", "qa", "security", "ui_ux", "chat", "analyzer"]
    provider_id: str
    model: str
    temperature: float
    system_prompt: str | None
    max_tokens: int | None
    enabled: bool
    created_at: datetime
    updated_at: datetime

@dataclass
class CLIModel:
    id: str
    name: str
    description: str

@dataclass
class CLITypeConfig:
    description: str
    models: list[CLIModel]

@dataclass
class CLIModelsConfig:
    gemini: CLITypeConfig | None
    claude: CLITypeConfig | None
    codex: CLITypeConfig | None
```

---

## Data Flow Visualization Contracts

API contracts for data flow visualizations (lineage, health, sync, schema, and query tracing).

See [DataFlowVisualization.md](./components/DataFlowVisualization.md) for full feature specification.

```python
@dataclass
class DataLineageNode:
    id: str
    type: Literal["source", "storage", "processor", "output"]
    label: str
    metadata: dict[str, Any]

@dataclass
class DataLineageEdge:
    id: str
    source: str
    target: str
    type: Literal["write", "read", "sync"]
    metadata: dict[str, Any]

@dataclass
class DataLineageGraph:
    nodes: list[DataLineageNode]
    edges: list[DataLineageEdge]

@dataclass
class DataLineageEvent:
    id: str
    edge_id: str
    event_type: str
    status: Literal["pending", "processing", "success", "failed"]
    occurred_at: datetime
    metadata: dict[str, Any] | None

@dataclass
class StorageHealth:
    """Health status for a storage backend."""
    backend: Literal["postgres", "lightrag", "redis", "temporal", "markdown"]
    status: Literal["healthy", "degraded", "down", "unknown"]
    record_count: int | None
    latency_ms: float | None
    last_query_at: datetime | None
    error: str | None

@dataclass
class SyncEvent:
    """Event in the sync outbox."""
    id: str
    event_type: str  # file_indexed, relationship_created, etc.
    entity_id: str
    status: Literal["pending", "processing", "success", "failed", "dead_letter"]
    target_backends: list[str]
    created_at: datetime
    processed_at: datetime | None
    error: str | None
    retry_count: int

@dataclass
class EntityLocationStatus:
    status: Literal["present", "missing", "stale", "pending"]
    version: str | None
    synced_at: datetime | None

@dataclass
class EntityLocation:
    """Where an entity exists across storage backends."""
    entity_id: str
    entity_type: str
    locations: dict[str, EntityLocationStatus]  # backend -> status

@dataclass
class PostgresColumn:
    name: str
    data_type: str
    nullable: bool
    is_primary_key: bool
    default: str | None

@dataclass
class PostgresTable:
    name: str
    row_count: int | None
    columns: list[PostgresColumn]

@dataclass
class PostgresForeignKey:
    table: str
    column: str
    ref_table: str
    ref_column: str

@dataclass
class PostgresSchema:
    tables: list[PostgresTable]
    foreign_keys: list[PostgresForeignKey]

@dataclass
class LightRAGEntityType:
    name: str
    count: int
    relationships: dict[str, int]

@dataclass
class LightRAGSchema:
    entity_types: list[LightRAGEntityType]

@dataclass
class WorkflowDataFlowStep:
    id: str
    name: str
    status: Literal["pending", "running", "completed", "failed"]
    started_at: datetime | None
    completed_at: datetime | None

@dataclass
class WorkflowStorageEdge:
    step_id: str
    backend: Literal["postgres", "lightrag", "redis", "temporal", "markdown"]
    action: Literal["read", "write"]

@dataclass
class WorkflowDataFlow:
    workflow_id: str
    steps: list[WorkflowDataFlowStep]
    storage_edges: list[WorkflowStorageEdge]

@dataclass
class StorageDiffEntry:
    entity_id: str
    entity_type: str
    versions: dict[str, str | None]
    status: Literal["in_sync", "stale", "missing"]
    last_synced_at: datetime | None

@dataclass
class QueryStage:
    name: str
    started_at: datetime | None
    completed_at: datetime | None
    latency_ms: float | None

@dataclass
class QueryBackendTrace:
    backend: str
    latency_ms: float
    result_count: int
    error: str | None

@dataclass
class QueryTrace:
    """Trace of a query across storage backends."""
    query_id: str
    query_text: str
    started_at: datetime
    completed_at: datetime | None
    total_latency_ms: float
    stages: list[QueryStage]
    backends_queried: list[QueryBackendTrace]
    result_count: int
```

### Data Flow API Endpoints

| Method | Endpoint                         | Description                       |
| ------ | -------------------------------- | --------------------------------- |
| GET    | `/api/system/health`             | All storage backends health       |
| GET    | `/api/system/health/:backend`    | Specific backend health           |
| GET    | `/api/system/lineage`            | Data lineage graph structure      |
| GET    | `/api/system/lineage/events`     | Recent lineage events             |
| GET    | `/api/sync/events`               | List sync events with status      |
| POST   | `/api/sync/events/:id/retry`     | Manual retry of failed event      |
| GET    | `/api/sync/diff`                 | Cross-storage diff view           |
| POST   | `/api/sync/diff/resolve`         | Bulk sync selected entities       |
| GET    | `/api/sync/diff/stats`           | Diff status summary               |
| GET    | `/api/entities/:id/locations`    | Where entity exists               |
| POST   | `/api/entities/:id/sync/:backend` | Trigger sync to a backend         |
| GET    | `/api/schema/postgres`           | Postgres tables and FKs           |
| GET    | `/api/schema/lightrag`           | LightRAG entity types             |
| POST   | `/api/schema/lightrag/query`     | Custom entity query               |
| GET    | `/api/workflows/:id/dataflow`    | Workflow data flow graph          |
| GET    | `/api/queries/trace/:id`         | Query execution trace             |
| GET    | `/api/queries/stats`             | Query performance summary         |

### WebSocket Endpoints

| Endpoint               | Description             |
| ---------------------- | ----------------------- |
| `WS /ws/system/lineage`| Live lineage event feed |
| `WS /ws/sync/events`   | Live sync event stream  |
| `WS /ws/workflows/:id` | Live workflow updates   |
| `WS /ws/queries`       | Live query stream       |

---

## Chat Contracts (Dashboard -> Storage)

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
    role: Literal["user", "assistant", "system"]
    content: str
    citations: list[str]
    created_at: datetime
```

---

## Error Codes

### Scanner Errors

| Code                     | Meaning                    |
| ------------------------ | -------------------------- |
| `SCAN_PERMISSION_DENIED` | Cannot read file/directory |
| `SCAN_FILE_TOO_LARGE`    | File exceeds size limit    |
| `SCAN_INVALID_ENCODING`  | File not UTF-8             |

### Parser Errors

| Code                         | Meaning                |
| ---------------------------- | ---------------------- |
| `PARSE_UNSUPPORTED_LANGUAGE` | Language not supported |
| `PARSE_SYNTAX_ERROR`         | File has syntax errors |
| `PARSE_TIMEOUT`              | Parsing took too long  |

### Storage Errors

| Code                        | Meaning              |
| --------------------------- | -------------------- |
| `STORAGE_CONNECTION_FAILED` | Cannot connect to DB |
| `STORAGE_WRITE_FAILED`      | Cannot write data    |
| `STORAGE_NOT_FOUND`         | Item not found       |

### MCP Errors

| Code                 | Meaning                |
| -------------------- | ---------------------- |
| `MCP_RATE_LIMITED`   | Too many requests      |
| `MCP_UNAUTHORIZED`   | Invalid/missing auth   |
| `MCP_INVALID_PARAMS` | Bad request parameters |

---

## Retry Semantics

### When to Retry

| Error Type      | Retry              | Max Attempts |
| --------------- | ------------------ | ------------ |
| Network timeout | Yes                | 3            |
| Rate limited    | Yes (with backoff) | 5            |
| Auth failed     | No                 | 1            |
| Not found       | No                 | 1            |
| Parse error     | No                 | 1            |

### Backoff Strategy

```python
def get_retry_delay(attempt: int) -> float:
    """Exponential backoff with jitter."""
    base = 1.0  # seconds
    max_delay = 30.0
    delay = min(base * (2 ** attempt), max_delay)
    jitter = random.uniform(0, 0.1 * delay)
    return delay + jitter
```
