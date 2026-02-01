# Postgres

Postgres is the primary source of truth for Docrunch metadata, tasks, reports,
LLM settings, and chat history.

## Responsibilities

- Structured metadata (files, relationships, tasks, reports)
- LLM provider, model, and specialist profile settings
- Chat sessions and messages
- Sync outbox tables (scan_runs, sync_events, storage_sync)

## Connection

Use SQLAlchemy 2.0 async with asyncpg.

```yaml
storage:
  postgres_url: postgresql://docrunch:docrunch@localhost:5430/docrunch
```

## Schema Overview

### Core Metadata

- files, relationships
- tasks, task_history, findings, decisions, completion_reports, blockers

### LLM Settings

- llm_providers (provider_type, config, enabled)
- llm_profiles (role -> provider/model, temperature, system_prompt)
- CLI Bridge models stored in `.docrunch/cli_models.json` (editable via UI/API)
- API keys are encrypted at rest and masked in the UI

### Chat

- chat_sessions
- chat_messages (role, content, citations)

### Sync Outbox

- scan_runs
- sync_events
- storage_sync

## Migrations

- Use Alembic for schema migrations.
- Run migrations on startup or via a dedicated command.

## Text Search

Fallback queries should use Postgres full-text search:

- Store a tsvector column per document/table
- Create GIN indexes for fast search

## Backup and Restore

Use standard Postgres tooling:

- pg_dump for backups
- pg_restore for restore

## Implementation Plan

1. Add SQLAlchemy async engine + session management
2. Define schema and migrations (Alembic)
3. Implement storage repository layer
4. Add tests for CRUD and search
5. Document operational setup (Docker, backups)
