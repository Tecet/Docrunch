# Agent Analytics

Performance tracking and insights for AI coding agents to optimize task assignment.

## Overview

Track which agents perform best at which task types to enable smarter auto-assignment.

Performance overview (last 30 days):

| Agent  | Tasks | Approval | Best at                | Avg time | Avg cost |
| ------ | ----- | -------- | ---------------------- | -------- | -------- |
| Claude | 45    | 92%      | docs, refactoring      | 12 min   | $0.18    |
| Gemini | 38    | 88%      | features, complex logic| 18 min   | $0.12    |
| Codex  | 22    | 85%      | tests, bug fixes       | 8 min    | $0.08    |

## Data Model

```python
@dataclass
class AgentMetrics:
    agent: str
    period_start: datetime
    period_end: datetime

    # Volume
    tasks_completed: int
    tasks_rejected: int

    # Quality
    approval_rate: float
    first_try_success_rate: float

    # Efficiency
    avg_completion_time: timedelta
    avg_tokens_used: int
    avg_cost: float

    # Specialization
    task_type_performance: dict[str, TypeMetrics]

@dataclass
class TypeMetrics:
    task_type: str
    count: int
    approval_rate: float
    avg_time: timedelta
    avg_cost: float
```

## Features

### 1. Smart Auto-Assignment

Use analytics to pick best agent:

```python
def select_best_agent(task: Task) -> str:
    """Select optimal agent based on historical performance."""
    metrics = get_all_agent_metrics()

    scores = {}
    for agent, m in metrics.items():
        type_perf = m.task_type_performance.get(task.type)
        if type_perf:
            scores[agent] = (
                type_perf.approval_rate * 0.5 +
                (1 / type_perf.avg_time.seconds) * 0.3 +
                (1 / type_perf.avg_cost) * 0.2
            )

    return max(scores, key=scores.get)
```

### 2. Performance Trends

Track improvement over time:

```python
def get_performance_trend(agent: str, days: int = 30) -> list[DailyMetrics]:
    """Get daily performance metrics for trending."""
```

### 3. Recommendations

Suggest improvements:

```python
def get_recommendations(agent: str) -> list[Recommendation]:
    """Get recommendations for agent improvement."""
    metrics = get_agent_metrics(agent)

    recs = []
    if metrics.task_type_performance['docs'].approval_rate < 0.8:
        recs.append(Recommendation(
            type="training",
            message="Consider providing more documentation examples"
        ))

    return recs
```

## Configuration

```yaml
# .docrunch/config.yaml
analytics:
  enabled: true
  retention_days: 90

  auto_assignment:
    enabled: true
    min_tasks_for_data: 10 # Need 10+ tasks before using data

  agents:
    claude:
      strengths: [docs, refactoring]
      default_for: [docs]
    gemini:
      strengths: [features, analysis]
      default_for: [feature]
    codex:
      strengths: [tests, quick_fixes]
      default_for: [test, bug_fix]
```

## MCP Tools

| Tool                 | Description                         |
| -------------------- | ----------------------------------- |
| `get_agent_stats`    | Get performance stats for agent     |
| `get_best_agent_for` | Get recommended agent for task type |
| `compare_agents`     | Compare agent performance           |

## Dashboard

- Agent leaderboard
- Performance charts over time
- Task type heatmaps
- Efficiency comparisons
