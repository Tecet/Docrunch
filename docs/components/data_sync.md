# Data Synchronization

This document defines how Docrunch keeps Postgres (source of truth), LightRAG (vectors), and Markdown docs in sync.

## Goals

- Make Postgres authoritative and durable.
- Keep LightRAG and Markdown derived and rebuildable.
- Ensure idempotent, retryable sync with clear failure visibility.
- Avoid blocking scans on downstream storage updates.

## Source of Truth and Projections

- Postgres is the only system of record for metadata and structured facts.
- LightRAG and Markdown are projections derived from Postgres.
- All writes land in Postgres first; projections are updated asynchronously.

## Sync Entities

Typical entities that produce projections:

- File metadata (files)
- Parsed elements (functions, classes, modules)
- Relationships (imports, calls, etc.)
- Tasks and decisions (if exposed as docs)
- Report events (progress, completion, findings, decisions, blockers)

## Sync Model

Use an outbox-style pattern so writes and sync events are in the same Postgres transaction.


## Execution Model

- Local/dev: in-process worker for simplicity.
- Hosted: Dramatiq workers using a Redis broker consume sync jobs.
- Sync events remain in Postgres and are enqueued for workers to process.

### Suggested Tables

```sql
-- Track scans for consistent grouping
CREATE TABLE scan_runs (
    id TEXT PRIMARY KEY,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    status TEXT
);

-- Outbox events for projections
CREATE TABLE sync_events (
    id TEXT PRIMARY KEY,
    scan_run_id TEXT,
    entity_type TEXT,
    entity_id TEXT,
    content_hash TEXT,
    action TEXT, -- upsert | delete
    created_at TIMESTAMP,
    FOREIGN KEY (scan_run_id) REFERENCES scan_runs(id)
);

-- Per-target sync state for idempotency and monitoring
CREATE TABLE storage_sync (
    id TEXT PRIMARY KEY,
    target TEXT, -- lightrag | markdown
    entity_type TEXT,
    entity_id TEXT,
    last_synced_hash TEXT,
    status TEXT, -- pending | ok | failed
    attempts INTEGER,
    last_error TEXT,
    updated_at TIMESTAMP,
    scan_run_id TEXT,
    UNIQUE (target, entity_type, entity_id)
);

-- Index for faster worker queries on pending/failed events
CREATE INDEX idx_storage_sync_pending ON storage_sync(target, status)
    WHERE status IN ('pending', 'failed');
```

## Sync Flow

1. Scan/parse/analyze generates entity records.
2. Write to Postgres (files, relationships, etc.) in a transaction.
3. In the same transaction, emit `sync_events` for each entity.
4. A worker (in-process or Redis-backed) processes events for each target (LightRAG, Markdown).
5. On success, update `storage_sync` with `last_synced_hash`.

## Idempotency Rules

- Each entity has a deterministic `content_hash`.
- For a target, if `last_synced_hash` matches the event hash, skip work.
- Events are safe to retry; processing is at-least-once.
- Workers should guard with `(target, entity_type, entity_id, content_hash)` to dedupe.

## Deletes

- Postgres records deletions as tombstones or explicit delete events.
- Workers must remove corresponding LightRAG documents and Markdown files.
- Deletions should be idempotent (missing doc is OK).

## Consistency Guarantees

- Postgres is strongly consistent and authoritative.
- LightRAG and Markdown are eventually consistent.
- The system should expose sync lag (e.g., last scan_run_id synced).

## Report Sync SLA (Defaults)

- Report events should reach derived stores within 60 seconds.
- Scan/index events should reach derived stores within 5 minutes.
- Report events are higher priority than scan events.
- SLA breach if report lag exceeds 2x target for more than 5 minutes.

## Error Policy (Defaults)

- Exponential backoff with jitter, starting at 2 seconds and capping at 120 seconds.
- Max retries: 5 attempts before marking failed.
- Failed events remain in Postgres and can be retried manually.
- Retryable: network timeouts, Redis outages, transient API errors.
- Non-retryable: validation errors, schema mismatch, missing entities.

## Failure Handling

- Retry with backoff, track attempts in `storage_sync`.
- Mark as failed after N retries; surface in CLI and dashboard.
- Provide a manual retry or rebuild command.

## Rebuild and Repair

- LightRAG and Markdown can be fully rebuilt from Postgres.
- Commands (planned):
  - `docrunch sync status`
  - `docrunch sync retry --failed`
  - `docrunch sync rebuild --vectors`
  - `docrunch sync rebuild --markdown`

## Ordering and Priority

- Postgres writes always precede projection updates.
- Markdown and LightRAG can be updated in parallel.
- Prioritize report events for query readiness.

## Example Worker Pseudocode

```python
class SyncWorker:
    def process_event(self, event: SyncEvent):
        if event.action == "delete":
            self.delete_projection(event)
            return

        if self.is_already_synced(event):
            return

        self.update_projection(event)
```

## Configuration (Draft)

```yaml
# .docrunch/config.yaml
storage:
  sync:
    enabled: true
    targets: [lightrag, markdown]
    max_retries: 5
    backoff_seconds: 2
    max_backoff_seconds: 120
    report_sla_seconds: 60
    scan_sla_seconds: 300
    poll_interval_seconds: 5
    prioritize: reports

jobs:
  backend: dramatiq
  broker: redis
  redis_url: redis://localhost:6370/0
  queue_name: docrunch
  poll_interval_seconds: 5
```

## Observability

- Track: events processed/sec, failed events, average lag per target.
- Surface a simple health view: last scan_run_id synced per target.
