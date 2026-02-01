# Webhook & Notifications

Event-driven notifications for task updates, alerts, and integrations.

## Overview

Send notifications and trigger webhooks when events occur in Docrunch.

Channels:
- Slack: #docrunch-alerts (connected)
- Email: dev@example.com (connected)
- Webhook: ci.example.com/hook (connected)

Recent notifications:
- Task #015 completed -> Slack, Email (2m ago)
- Review needed: Task #016 -> Slack (5m ago)
- Budget alert: 80% used -> Email (15m ago)

## Supported Channels

| Channel | Use Case           |
| ------- | ------------------ |
| Slack   | Team notifications |
| Discord | Team notifications |
| Email   | Personal alerts    |
| Webhook | CI/CD integration  |
| Teams   | Enterprise teams   |

## Events

| Event                | Description              |
| -------------------- | ------------------------ |
| `task.created`       | New task created         |
| `task.assigned`      | Task assigned to agent   |
| `task.started`       | Agent started work       |
| `task.completed`     | Task finished            |
| `task.review_needed` | Awaiting review          |
| `task.approved`      | Review passed            |
| `task.rejected`      | Review failed            |
| `epic.completed`     | All subtasks done        |
| `cost.alert`         | Budget threshold reached |
| `scan.completed`     | Repository scan finished |
| `error.occurred`     | Error during operation   |

## Configuration

```yaml
# .docrunch/config.yaml
notifications:
  enabled: true

  channels:
    slack:
      enabled: true
      webhook_url_env: SLACK_WEBHOOK_URL
      channel: "#docrunch-alerts"

    discord:
      enabled: false
      webhook_url_env: DISCORD_WEBHOOK_URL

    email:
      enabled: true
      smtp:
        host: smtp.gmail.com
        port: 587
        user_env: SMTP_USER
        password_env: SMTP_PASSWORD
      recipients:
        - dev@example.com

    webhook:
      enabled: true
      endpoints:
        - url: https://ci.example.com/docrunch-hook
          secret_env: WEBHOOK_SECRET
          events: [task.completed, task.rejected]

  rules:
    - event: task.completed
      priority: [high, critical]
      channels: [slack, email]

    - event: task.review_needed
      channels: [slack]

    - event: cost.alert
      channels: [email]

    - event: task.*
      priority: low
      channels: [] # No notification for low priority
```

## Message Templates

```yaml
# .docrunch/notification_templates.yaml
templates:
  task.completed:
    slack:
      text: "Task completed: {task.title}"
      blocks:
        - type: section
          text: |
            *{task.title}* completed by {task.agent}

            Tokens: {task.tokens_used}
            Cost: ${task.cost}
            Time: {task.duration}

    email:
      subject: "[Docrunch] Task Completed: {task.title}"
      body: |
        Task {task.id} has been completed.

        Summary: {task.completion_report.summary}
        Files changed: {task.completion_report.changes}

  task.review_needed:
    slack:
      text: "Review needed: {task.title}"
      blocks:
        - type: actions
          elements:
            - type: button
              text: "Review Now"
              url: "{dashboard_url}/tasks/{task.id}"
```

## Webhook Payload

```json
{
  "event": "task.completed",
  "timestamp": "2024-01-15T10:30:00Z",
  "task": {
    "id": "task-015",
    "title": "Add user auth",
    "type": "feature",
    "agent": "claude",
    "status": "completed",
    "completion_report": {
      "summary": "Implemented JWT authentication...",
      "changes": ["auth.py", "users.py"],
      "tests_passed": true
    }
  },
  "signature": "sha256=..."
}
```

## Features

### 1. Rule-Based Routing

```python
def route_notification(event: Event):
    """Route event to appropriate channels."""
    for rule in get_notification_rules():
        if rule.matches(event):
            for channel in rule.channels:
                send_notification(channel, event)
```

### 2. Rate Limiting

```yaml
rate_limits:
  slack:
    max_per_minute: 10
    batch_similar: true
  email:
    max_per_hour: 20
```

### 3. Digest Mode

Batch notifications into periodic digests:

```yaml
digest:
  enabled: true
  frequency: hourly # hourly, daily
  channels: [email]
```

## CLI Commands

```bash
# Test notification
docrunch notify test --channel slack

# List notification history
docrunch notify history

# Configure channel
docrunch notify config slack --webhook-url <url>
```

## MCP Tools

| Tool                       | Description              |
| -------------------------- | ------------------------ |
| `send_notification`        | Send manual notification |
| `get_notification_history` | View past notifications  |

## Dashboard

- Notification log
- Channel status
- Rule configuration
- Test notifications
