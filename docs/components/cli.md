# Command Line Interface

The Docrunch CLI provides commands for scanning, querying, and managing repositories.

## Installation

```bash
pip install docrunch
```

Or install from source:

```bash
cd D:\Workspaces\Docrunch
pip install -e .
```

## Commands Overview

### MVP (available)

| Command          | Description                         |
| ---------------- | ----------------------------------- |
| `docrunch init`  | Initialize Docrunch in a repository |
| `docrunch scan`  | Full repository scan                |
| `docrunch serve` | Start MCP server / REST API         |
| `docrunch query` | Query documentation                 |

### Planned (post-MVP)

| Command              | Description                   | Phase     |
| -------------------- | ----------------------------- | --------- |
| `docrunch watch`     | Real-time file watching       | Phase 2+  |
| `docrunch sync`      | Storage sync management       | Phase 1B+ |
| `docrunch worker`    | Run background workers        | Phase 2+  |
| `docrunch scheduler` | Run scheduled jobs (cron)     | Phase 2+  |
| `docrunch config`    | Manage configuration          | Phase 2+  |
| `docrunch task`      | Manage tasks                  | Phase 3+  |
| `docrunch cost`      | LLM cost tracking and budgets | Phase 5+  |
| `docrunch epic`      | Epic planning and status      | Phase 5+  |
| `docrunch notify`    | Notifications and webhooks    | Phase 5+  |
| `docrunch git`       | Git integration tools         | Phase 5+  |
| `docrunch rollback`  | Snapshots and recovery        | Phase 5+  |

Note: Planned commands are documented for design alignment and will be implemented in later phases.

---

## docrunch init

Initialize Docrunch in a repository.

```bash
docrunch init [OPTIONS] [PATH]
```

### Options

| Option       | Description                                  |
| ------------ | -------------------------------------------- |
| `--path`     | Repository path (default: current directory) |
| `--template` | Config template (minimal, standard, full)    |

### Example

```bash
cd my-project
docrunch init --template standard
```

### Output

```
.docrunch/
- config.yaml
- task_templates.yaml
- review_rules.yaml
- ignore.yaml
```

---

## docrunch scan

Scan repository and generate documentation.

```bash
docrunch scan [OPTIONS] [PATH]
```

### Options

| Option      | Description       |
| ----------- | ----------------- |
| `--path`    | Repository path   |
| `--full`    | Force full rescan |
| `--output`  | Output directory  |
| `--offline` | Skip Postgres/LLM and cache metadata locally (`.docrunch/offline_files.json`) |
| `--quiet`   | Minimal output    |
| `--verbose` | Detailed output   |

### Example

```bash
docrunch scan --verbose
```

```bash
docrunch scan --offline --output docrunch-docs-offline
```

### Output

```
Scanning repository...
  Files found: 156
  Languages: python (45), typescript (89), css (22)

Parsing...
  Functions: 234
  Classes: 45
  Components: 67

Analyzing relationships...
  Imports: 456
  Dependencies: 123

Generating documentation...
  Modules: 12
  Schemas: 5

Writing to docrunch-docs/...
Done! Documentation available at ./docrunch-docs/
```

`--offline` mode skips Postgres and any sync/outbox/queue enqueues, writing metadata to `.docrunch/offline_files.json` while still generating Markdown output.

---

## docrunch watch

Start real-time file watching with incremental updates.

```bash
docrunch watch [OPTIONS] [PATH]
```

### Options

| Option       | Description                        |
| ------------ | ---------------------------------- |
| `--path`     | Repository path                    |
| `--debounce` | Debounce time in ms (default: 500) |
| `--ignore`   | Additional ignore patterns         |

### Example

```bash
docrunch watch --debounce 1000
```

### Output

```
Watching for changes...
[14:32:01] Modified: src/auth/login.py
  Re-indexing...
  Updated: 3 relationships, 1 function
[14:33:45] Created: src/api/users.py
  Indexing new file...
  Added: 5 functions, 2 exports
```

---

## docrunch serve

Start MCP server for AI agent connections and/or REST API for dashboard.

```bash
docrunch serve [OPTIONS]
```

### Options

| Option    | Description                            |
| --------- | -------------------------------------- |
| `--http`  | Enable REST API server (HTTP)          |
| `--host`  | HTTP host address (default: localhost) |
| `--port`  | HTTP server port (default: 3333)       |
| `--stdio` | Enable MCP over stdio (CLI agents)     |

### Unified Server Startup Guide

Choose the right combination based on your use case:

| Use Case                                    | Command                             | Ports                     |
| ------------------------------------------- | ----------------------------------- | ------------------------- |
| **Dashboard only**                          | `docrunch serve --http`             | REST API on :3333         |
| **CLI agents only** (Claude, Gemini, Codex) | `docrunch serve --stdio`            | stdio (no network)        |
| **Both dashboard + CLI agents**             | `docrunch serve --http --stdio`     | REST API on :3333 + stdio |
| **Custom port**                             | `docrunch serve --http --port 8080` | REST API on :8080         |

### Example Startup Commands

```bash
# Development: REST API for dashboard UI
docrunch serve --http --port 3333
# Access dashboard at http://localhost:5173 (Vite dev server)
# API available at http://localhost:3333/api

# Production: MCP stdio only for CLI agents
docrunch serve --stdio
# Configure in Claude CLI: "command": "docrunch", "args": ["serve", "--stdio"]

# Full stack: Both REST and MCP stdio
docrunch serve --http --stdio --port 3333
# Supports both dashboard and CLI agent connections simultaneously
```

### Transport Configuration

For MCP over HTTP (optional, instead of stdio), configure in `.docrunch/config.yaml`:

```yaml
server:
  mcp:
    transport: http # Change from stdio to http
    host: localhost
    port: 3334 # Separate from REST API port
```

Then start with: `docrunch serve --http` (MCP will use HTTP transport on port 3334)

---

## docrunch query

Query documentation from command line.

```bash
docrunch query [OPTIONS] QUERY
```

### Options

| Option     | Description                          |
| ---------- | ------------------------------------ |
| `--limit`  | Max results (default: 5)             |
| `--format` | Output format (text, json, markdown) |

### Example

```bash
docrunch query "How does authentication work?"
```

### Output

```
Found 3 relevant results:

1. modules/auth.md (score: 0.92)
   The auth module handles JWT-based authentication...

2. functions/verify_token.md (score: 0.87)
   Verifies JWT token and returns user payload...

3. patterns/auth_middleware.md (score: 0.81)
   Authentication middleware pattern used across API routes...
```

---

## docrunch task

Manage tasks from command line.

```bash
docrunch task [SUBCOMMAND]
```

### Subcommands

#### list

```bash
docrunch task list [--status STATUS] [--assignee AGENT]
```

#### create

```bash
docrunch task create --type feature --title "Add user profile" --description "..."
```

#### show

```bash
docrunch task show TASK_ID
```

#### assign

```bash
docrunch task assign TASK_ID --to gemini
```

### Example

```bash
$ docrunch task list --status todo

ID         TYPE     PRIORITY  TITLE
task-001   feature  high      Add user profile
task-002   bug_fix  medium    Fix login timeout
task-003   docs     low       Update API docs
```

---

## docrunch config

Manage configuration.

```bash
docrunch config [SUBCOMMAND]
```

### Subcommands

#### show

```bash
docrunch config show
```

#### set

```bash
docrunch config set llm.default_provider anthropic
```

#### validate

```bash
docrunch config validate
```

---

## docrunch sync

Manage storage synchronization between Postgres, LightRAG, and Markdown.

```bash
docrunch sync [SUBCOMMAND]
```

### Subcommands

#### status

Show sync status and any failed events.

```bash
docrunch sync status
```

**Output:**

```
Storage Sync Status
-------------------
LightRAG:  OK (last sync: 2 min ago)
Markdown:  OK (last sync: 2 min ago)
Pending:   0
Failed:    0
```

#### retry

Retry failed sync events.

```bash
docrunch sync retry --failed
docrunch sync retry --all
```

#### rebuild

Rebuild derived stores from Postgres source of truth.

```bash
docrunch sync rebuild --vectors    # Rebuild LightRAG
docrunch sync rebuild --markdown   # Rebuild Markdown
docrunch sync rebuild --all        # Rebuild everything
```

---

## Global Options

Available for all commands:

| Option      | Description             |
| ----------- | ----------------------- |
| `--help`    | Show help message       |
| `--version` | Show version            |
| `--config`  | Custom config file path |
| `--debug`   | Enable debug logging    |

---

## Implementation

```
docrunch/cli/
- __init__.py
- main.py          # Entry point, Typer app
- init.py          # init command
- scan.py          # scan command
- watch.py         # watch command
- serve.py         # serve command
- query.py         # query command
- task.py          # task subcommands
- config.py        # config subcommands
```

### Entry Point (main.py)

```python
import typer
from .init import init_cmd
from .scan import scan_cmd
from .watch import watch_cmd
from .serve import serve_cmd
from .query import query_cmd
from .task import task_app
from .config import config_app

app = typer.Typer(
    name="docrunch",
    help="Repository Intelligence & AI Orchestration System"
)

app.command("init")(init_cmd)
app.command("scan")(scan_cmd)
app.command("watch")(watch_cmd)
app.command("serve")(serve_cmd)
app.command("query")(query_cmd)
app.add_typer(task_app, name="task")
app.add_typer(config_app, name="config")

if __name__ == "__main__":
    app()
```

### pyproject.toml Entry

```toml
[project.scripts]
docrunch = "docrunch.cli.main:app"
```
