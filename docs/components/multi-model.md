# Multi-Model Collaboration

Enable multiple AI agents to work on subtasks of the same feature simultaneously.

## Overview

Break complex tasks into subtasks and assign different agents to each, enabling parallel AI development.

Epic: Build Authentication System
- Status: 3/4 subtasks complete

Subtasks:
- Database Schema (Gemini) - DONE
- API Routes (Claude) - DONE
- Frontend Forms (Claude) - WORKING
- Tests (Codex) - WAITING
  - Depends on: API Routes, Frontend

Sync points:
- Schema types shared with API Routes - DONE
- API contract shared with Frontend - DONE

## Data Model

```python
@dataclass
class Epic:
    id: str
    title: str
    description: str

    subtasks: list[str]  # Task IDs
    sync_points: list[SyncPoint]

    status: EpicStatus
    progress: float  # 0-100

@dataclass
class SyncPoint:
    id: str
    epic_id: str
    description: str

    producer_task: str
    consumer_tasks: list[str]

    artifact_type: str  # types, contract, schema
    artifact_content: str

    status: str  # pending, produced, consumed
```

## Features

### 1. Epic Planning

Break complex work into coordinated subtasks:

```python
def plan_epic(description: str) -> Epic:
    """Use LLM to plan epic with subtasks."""
    plan = llm.generate(f"""
        Break this feature into subtasks for parallel development:
        {description}

        Consider:
        - Database changes first
        - API next
        - Frontend after API contract defined
        - Tests last (need interfaces)

        Return subtasks with dependencies.
    """)

    return create_epic_from_plan(plan)
```

### 2. Agent Assignment by Strength

Assign subtasks based on analytics:

```python
def assign_epic_subtasks(epic: Epic) -> dict[str, str]:
    """Assign subtasks to best agents."""
    assignments = {}

    for task_id in epic.subtasks:
        task = get_task(task_id)
        best_agent = analytics.get_best_agent_for(task.type)
        assignments[task_id] = best_agent

    return assignments
```

### 3. Sync Points

Share artifacts between agents:

```python
def produce_sync_point(task_id: str, artifact: str):
    """Produce artifact for other tasks to consume."""
    sync = get_sync_point_for_producer(task_id)
    sync.artifact_content = artifact
    sync.status = "produced"

    # Notify waiting tasks
    for consumer_id in sync.consumer_tasks:
        notify_task_unblocked(consumer_id)

def consume_sync_point(task_id: str, sync_id: str) -> str:
    """Get artifact from sync point."""
    sync = get_sync_point(sync_id)
    return sync.artifact_content
```

### 4. Conflict Resolution

Handle conflicts between parallel work:

```python
def check_conflicts(epic_id: str) -> list[Conflict]:
    """Check for conflicts between subtask outputs."""
    epic = get_epic(epic_id)

    conflicts = []
    for task1, task2 in combinations(epic.subtasks, 2):
        overlap = find_file_overlap(task1, task2)
        if overlap:
            conflicts.append(Conflict(
                tasks=[task1, task2],
                files=overlap
            ))

    return conflicts
```

## Configuration

```yaml
# .docrunch/config.yaml
collaboration:
  enabled: true

  planning:
    auto_split: true # Auto-split large tasks
    min_subtasks: 2
    max_subtasks: 8

  sync_points:
    auto_detect: true
    types: [types, schemas, contracts, interfaces]

  conflict_resolution:
    mode: merge_review # auto_merge, merge_review, manual
```

## MCP Tools

| Tool                  | Description                            |
| --------------------- | -------------------------------------- |
| `create_epic`         | Create epic with subtasks              |
| `get_epic_status`     | Get epic progress                      |
| `produce_artifact`    | Produce sync point artifact            |
| `consume_artifact`    | Get sync point artifact                |
| `get_subtask_context` | Get context including shared artifacts |

## CLI Commands

```bash
# Create epic from description
docrunch epic create "Build user authentication"

# View epic status
docrunch epic status epic-001

# Assign subtasks
docrunch epic assign epic-001
```

## Dashboard

- Epic overview with subtask tree
- Real-time progress tracking
- Sync point visualization
- Conflict alerts
- Agent workload view
