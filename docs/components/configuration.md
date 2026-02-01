# Configuration

Docrunch uses a dedicated `.docrunch/` directory for configuration files.

## Directory Structure

```
.docrunch/
- config.yaml           # Main configuration
- cli_models.json       # CLI Bridge models registry
- task_templates.yaml   # Task type templates
- review_rules.yaml     # Review workflow rules
- ignore.yaml           # Additional ignore patterns
```

---

## Main Configuration (`config.yaml`)

```yaml
# .docrunch/config.yaml

# Project metadata
project:
  name: "MyProject"
  description: "Project description for context"
  version: "1.0.0"

# Scanning configuration
scanning:
  # Languages to parse
  languages:
    - python
    - typescript
    - javascript
    - sql
    - css

  # Directories to ignore (in addition to .gitignore)
  ignore:
    - node_modules/
    - __pycache__/
    - .git/
    - dist/
    - build/
    - .venv/
    - "*.min.js"
    - "*.map"

  # File size limits
  max_file_size_kb: 500

  # Concurrency
  parallel_workers: 4

# LLM configuration
llm:
  settings_source: database # stored in Postgres and editable in UI
  default_profile: manager
  allow_ui_updates: true
  bootstrap_from_env: false # optional first-run seed

# LLM providers and specialist profiles live in Postgres and are managed in the dashboard.
# CLI Bridge models live in .docrunch/cli_models.json.

# Storage configuration
storage:
  # Vector database
  vector_db: lightrag

  # LightRAG settings
  lightrag:
    base_url: http://localhost:9621
    embedding_model: text-embedding-3-small
    chunk_size: 1000
    chunk_overlap: 200

  # Postgres connection (source of truth)
  postgres_url: postgresql://docrunch:docrunch@localhost:5430/docrunch

  # Markdown output directory
  docs_output: ./docrunch-docs/

  # Storage sync
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

# Background jobs / queue
jobs:
  backend: dramatiq # dramatiq or inprocess
  broker: redis
  redis_url: redis://localhost:6370/0
  queue_name: docrunch
  poll_interval_seconds: 5
  scheduler:
    enabled: true
    driver: dramatiq_cron
    scan_cron: "0 */6 * * *"
    sync_cron: "*/5 * * * *"

# File watcher configuration (Phase 2+)
watch:
  enabled: false # Disabled by default until Phase 2
  debounce_ms: 500

  # Additional patterns to watch
  include:
    - "*.py"
    - "*.ts"
    - "*.tsx"
    - "*.js"
    - "*.jsx"
    - "*.sql"
    - "*.css"
    - "*.scss"

# Server configuration (REST + MCP)
server:
  # REST API (for dashboard + integrations)
  http:
    enabled: true
    host: localhost
    port: 3333
    base_path: /api
    cors_origins:
      - http://localhost:5173
      - http://localhost:3000

  # MCP settings
  mcp:
    enabled: true
    transport: stdio # stdio or http
    host: localhost # used only for http transport
    port: 3334 # use a separate port from REST

    # Rate limiting
    rate_limit:
      queries_per_minute: 60
      writes_per_minute: 30

# Dashboard configuration
dashboard:
  port: 5173
  theme: dark
```

---

## Security and Auth (Draft)

MCP auth and rate limiting live under `server.mcp`:

```yaml
server:
  mcp:
    auth:
      enabled: true
      tokens:
        - name: claude-agent
          token_env: DOCRUNCH_AGENT_TOKEN
          permissions: [read, write]
```

Dashboard auth endpoints are planned for Phase 4+ (see `docs/components/security.md`).

---

## Advanced Feature Settings (Draft)

These keys are used by Phase 4+ features and are documented in component docs:

```yaml
session:
  enabled: true
  retention_days: 90

analytics:
  enabled: true
  retention_days: 90

cost:
  alerts:
    daily_limit: 10.0

notifications:
  enabled: true

git:
  enabled: true

rollback:
  enabled: true

collaboration:
  enabled: true
```

See:

- `docs/components/session-memory.md`
- `docs/components/agent-analytics.md`
- `docs/components/cost-tracking.md`
- `docs/components/webhooks.md`
- `docs/components/git-integration.md`
- `docs/components/rollback-system.md`
- `docs/components/multi-model.md`

---

## Task Templates (`task_templates.yaml`)

```yaml
# .docrunch/task_templates.yaml

templates:
  bug_fix:
    description: "Fix a reported bug or issue"
    guardrails:
      - "Do not change API signatures without approval"
      - "Write regression test for the bug"
      - "Document the fix in commit message"
    acceptance_criteria:
      - "Bug no longer reproduces"
      - "All existing tests pass"
      - "New regression test added"
    review_mode:
      low: auto
      medium: manager
      high: human
      critical: human
    context_hints:
      - "Include related error logs"
      - "Find similar past bugs"

  feature:
    description: "Implement new functionality"
    guardrails:
      - "Follow existing architectural patterns"
      - "Update documentation for new features"
      - "Add comprehensive tests"
      - "Consider backward compatibility"
    acceptance_criteria:
      - "Feature works as specified"
      - "All tests pass"
      - "Documentation updated"
      - "No performance regression"
    review_mode: human
    context_hints:
      - "Include relevant patterns"
      - "Include related schemas"

  refactor:
    description: "Improve code without changing behavior"
    guardrails:
      - "No behavior changes allowed"
      - "All existing tests must pass"
      - "Maintain API compatibility"
    acceptance_criteria:
      - "Code is cleaner/faster/simpler"
      - "No functionality changes"
      - "All tests pass"
      - "Performance maintained or improved"
    review_mode: manager

  docs:
    description: "Update documentation"
    guardrails:
      - "Keep consistent style"
      - "Include code examples where helpful"
    acceptance_criteria:
      - "Documentation is accurate"
      - "Examples work correctly"
    review_mode: auto

  test:
    description: "Add or fix tests"
    guardrails:
      - "Follow existing test patterns"
      - "Test edge cases"
    acceptance_criteria:
      - "Tests are meaningful"
      - "Coverage improved"
    review_mode: auto

  security:
    description: "Security-related changes"
    guardrails:
      - "Never expose sensitive data in logs"
      - "Follow OWASP guidelines"
      - "Document security implications"
    acceptance_criteria:
      - "Vulnerability addressed"
      - "Security tests added"
      - "No new vulnerabilities introduced"
    review_mode: human # Always human review
```

---

## Review Rules (`review_rules.yaml`)

```yaml
# .docrunch/review_rules.yaml

review:
  # Default rules by priority
  rules:
    - priority: critical
      mode: human
      notify: ["admin@example.com"]

    - priority: high
      mode: manager
      escalate_after_hours: 24

    - priority: medium
      mode: auto
      conditions:
        - tests_passed: true
        - no_schema_changes: true
        - files_changed_max: 10
      fallback: manager

    - priority: low
      mode: auto
      conditions:
        - tests_passed: true

  # Task type overrides
  type_overrides:
    security:
      mode: human
      required_reviewers: 2

    feature:
      mode: human
      require_docs_update: true

  # Auto-approval conditions
  auto_approve:
    max_files_changed: 5
    max_lines_changed: 200
    require_tests: true
    forbidden_patterns:
      - "password"
      - "secret"
      - "api_key"
      - "DROP TABLE"

  # Manager AI configuration
  manager:
    provider: anthropic
    model: claude-3-5-sonnet-20241022
    review_prompt: |
      Review this task completion and determine if it should be approved.
      Check:
      1. Acceptance criteria met
      2. Guardrails followed
      3. Code quality acceptable
      4. Tests adequate
```

---

## Ignore Patterns (`ignore.yaml`)

```yaml
# .docrunch/ignore.yaml

# Additional patterns beyond .gitignore
patterns:
  # Build artifacts
  - dist/
  - build/
  - out/
  - .next/

  # Dependencies
  - node_modules/
  - vendor/
  - .venv/
  - venv/

  # IDE/Editor
  - .idea/
  - .vscode/
  - "*.swp"
  - "*.swo"

  # Generated files
  - "*.min.js"
  - "*.min.css"
  - "*.map"
  - "*.d.ts" # Keep if you want type definitions

  # Test artifacts
  - coverage/
  - .pytest_cache/
  - __pycache__/
  - "*.pyc"

  # Logs
  - "*.log"
  - logs/

  # Large files
  - "*.zip"
  - "*.tar.gz"
  - "*.pdf"

  # Docrunch internals
  - .docrunch/metadata.db
  - .docrunch/cache/
```

---

## Environment Variables

| Variable            | Description             |
| ------------------- | ----------------------- |
| `OPENAI_API_KEY`    | OpenAI API key          |
| `ANTHROPIC_API_KEY` | Anthropic API key       |
| `CLI_PROXY_URL`     | Default CLI Bridge proxy URL |
| `CLI_PROXY_API_KEY` | CLI Bridge proxy API key |
| `DOCRUNCH_CONFIG`   | Custom config file path |
| `DOCRUNCH_DEBUG`    | Enable debug logging    |

## Configuration Precedence

1. Command-line flags (highest)
2. Environment variables
3. `.docrunch/config.yaml`
4. Default values (lowest)

## Validation

```bash
# Validate configuration
docrunch config validate

# Show effective configuration
docrunch config show
```
