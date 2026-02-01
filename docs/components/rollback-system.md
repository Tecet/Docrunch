# Rollback System

Snapshot and restore documentation, vectors, and task states for safe recovery.

## Overview

The Rollback System creates snapshots before major operations, enabling one-click recovery if something goes wrong.

- Current state: v42
- Snapshots:
  - v42: task-018 completion (current) - 1h ago
  - v41: task-017 completion - 3h ago
  - v40: Manual scan - 6h ago
  - v39: task-016 completion - 12h ago
- Actions: rollback to v41, compare v41 vs v42, delete old

## Data Model

```python
@dataclass
class Snapshot:
    id: str
    version: int
    created_at: datetime
    trigger: str  # task_completion, manual, scan, auto

    # What's included
    includes_docs: bool
    includes_vectors: bool
    includes_tasks: bool

    # Reference
    task_id: str | None
    description: str

    # Storage
    docs_path: Path
    postgres_backup_path: Path
    vector_backup_path: Path

    # Metadata
    size_bytes: int
    files_count: int
```

## Features

### 1. Automatic Snapshots

Create snapshot before risky operations:

```python
class RollbackSystem:
    def create_snapshot(self, trigger: str, task_id: str = None) -> Snapshot:
        """Create full system snapshot."""
        snapshot = Snapshot(
            id=generate_id(),
            version=self.next_version(),
            trigger=trigger,
            task_id=task_id
        )

        # Backup docs
        snapshot.docs_path = self.backup_docs()

        # Backup Postgres
        snapshot.postgres_backup_path = self.backup_postgres()

        # Backup vectors
        snapshot.vector_backup_path = self.backup_vectors()

        return snapshot
```

### 2. One-Click Rollback

```python
def rollback_to(self, version: int):
    """Rollback entire system to snapshot version."""
    snapshot = self.get_snapshot(version)

    # Restore docs
    self.restore_docs(snapshot.docs_path)

    # Restore Postgres
    self.restore_postgres(snapshot.postgres_backup_path)

    # Restore vectors
    self.restore_vectors(snapshot.vector_backup_path)

    # Log rollback
    self.log_rollback(snapshot)
```

### 3. Diff Comparison

Compare snapshots to see what changed:

```python
def compare_snapshots(v1: int, v2: int) -> SnapshotDiff:
    """Compare two snapshot versions."""
    s1, s2 = get_snapshot(v1), get_snapshot(v2)

    return SnapshotDiff(
        docs_changed=diff_docs(s1.docs_path, s2.docs_path),
        tasks_changed=diff_tasks(s1, s2),
        vectors_changed=count_vector_changes(s1, s2)
    )
```

### 4. Selective Restore

Restore only specific components:

```python
def restore_selective(
    version: int,
    restore_docs: bool = True,
    restore_tasks: bool = True,
    restore_vectors: bool = True
):
    """Restore specific components from snapshot."""
```

## Configuration

```yaml
# .docrunch/config.yaml
rollback:
  enabled: true

  auto_snapshot:
    on_task_complete: true
    on_scan: true
    before_delete: true

  retention:
    max_snapshots: 50
    max_age_days: 30
    keep_minimum: 5

  storage:
    path: .docrunch/snapshots/
    compress: true
```

## CLI Commands

```bash
# List snapshots
docrunch rollback list

# Create manual snapshot
docrunch rollback create --description "Before major refactor"

# Rollback to version
docrunch rollback to 41

# Compare versions
docrunch rollback diff 41 42

# Clean old snapshots
docrunch rollback clean --older-than 30d
```

## MCP Tools

| Tool                | Description              |
| ------------------- | ------------------------ |
| `create_snapshot`   | Create manual snapshot   |
| `list_snapshots`    | List available snapshots |
| `rollback_to`       | Rollback to version      |
| `compare_snapshots` | Compare two versions     |

## Dashboard

- Snapshot timeline
- Diff viewer
- One-click rollback button
- Storage usage
