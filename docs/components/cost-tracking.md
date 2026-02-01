# Cost Tracking

Monitor and analyze LLM token usage and costs per task, agent, and project.

## Overview

Track every LLM API call to understand costs, optimize prompts, and budget AI development work.

- Today: $4.52
- This Week: $28.30
- Month: $142.00

By Agent:
- Claude: $2.15 (48%)
- Gemini: $1.89 (42%)
- OpenAI: $0.48 (10%)

By Task Type:
- Feature: $18.50 (65%)
- Bug Fix: $5.20 (18%)
- Docs: $4.60 (17%)

Recent Tasks:

| Task     | Title         | Tokens | Cost |
| -------- | ------------- | ------ | ---- |
| task-015 | Add Auth      | 45,230 | $0.23 |
| task-014 | Fix Login     | 12,100 | $0.06 |
| task-013 | Refactor API  | 89,500 | $0.45 |

## Data Model

```python
@dataclass
class LLMCall:
    id: str
    timestamp: datetime

    # Context
    task_id: str | None
    session_id: str | None
    agent: str

    # Request
    provider: str  # openai, anthropic, xai, google, ollama, cli_bridge
    model: str
    prompt_tokens: int

    # Response
    completion_tokens: int
    total_tokens: int

    # Cost
    cost_usd: float

    # Metadata
    purpose: str  # analyze, summarize, generate, explain
    cached: bool

@dataclass
class CostReport:
    period_start: datetime
    period_end: datetime

    total_cost: float
    total_tokens: int

    by_agent: dict[str, float]
    by_task_type: dict[str, float]
    by_provider: dict[str, float]
    by_purpose: dict[str, float]

    efficiency_score: float  # Compared to baseline
```

## Features

### 1. Real-time Tracking

Every LLM call is logged:

```python
class CostTracker:
    def track_call(self, call: LLMCall):
        """Log LLM API call with cost."""
        # Calculate cost based on provider pricing
        call.cost_usd = self.calculate_cost(
            call.provider,
            call.model,
            call.prompt_tokens,
            call.completion_tokens
        )
        self.db.insert(call)

    def calculate_cost(self, provider, model, prompt, completion) -> float:
        rates = self.pricing[provider][model]
        return (
            (prompt / 1000) * rates['input'] +
            (completion / 1000) * rates['output']
        )
```

### 2. Budget Alerts

Set spending limits:

```yaml
# .docrunch/config.yaml
cost:
  alerts:
    daily_limit: 10.00
    weekly_limit: 50.00
    monthly_limit: 200.00

  notifications:
    - type: email
      threshold: 80% # Alert at 80% of limit
    - type: slack
      threshold: 100%
```

### 3. Cost Optimization Suggestions

```python
def analyze_efficiency(task_id: str) -> EfficiencyReport:
    """Analyze if task used tokens efficiently."""
    calls = get_calls_for_task(task_id)

    return EfficiencyReport(
        total_tokens=sum(c.total_tokens for c in calls),
        avg_task_tokens=get_avg_tokens_for_type(task.type),
        suggestions=[
            "Consider caching repeated queries",
            "Use smaller model for simple tasks"
        ]
    )
```

## Pricing Configuration

```yaml
# .docrunch/pricing.yaml
providers:
  openai:
    gpt-4o:
      input: 0.0025 # per 1K tokens
      output: 0.01
    gpt-4o-mini:
      input: 0.00015
      output: 0.0006

  anthropic:
    claude-3-5-sonnet:
      input: 0.003
      output: 0.015
    claude-3-haiku:
      input: 0.00025
      output: 0.00125

  ollama:
    "*":
      input: 0
      output: 0

  cli_bridge:
    "*":
      input: 0
      output: 0

  cli_bridge:
    "*":
      input: 0
      output: 0
```

## MCP Tools

| Tool                | Description                 |
| ------------------- | --------------------------- |
| `get_task_cost`     | Get cost breakdown for task |
| `get_cost_report`   | Get cost report for period  |
| `get_budget_status` | Check remaining budget      |

## CLI Commands

```bash
# View today's costs
docrunch cost today

# View cost report
docrunch cost report --period week

# Set budget
docrunch cost budget --daily 10
```

## Dashboard

- Real-time cost ticker
- Historical charts
- Cost per task breakdown
- Budget progress bars
- Optimization suggestions
