# MVP Definition

## Minimum Viable Product Scope

The MVP delivers **core value**: AI agents can query context and report back via MCP, keeping the knowledge base current.

### What's IN the System (Current State)

- **Core Engine**: Scanner, AST parser (Python, TS, JS, TSX), Relationship Analyzer
- **Storage**: Postgres metadata, LightRAG vector graph, Markdown output
- **Integration**: Bedirectional MCP server (query + report tools)
- **LLM Support**: OpenAI, Anthropic, and **CLI Bridge** (Gemini/Claude/Codex CLIs)
- **UI/UX**: **React Dashboard**, WebSocket event streams, Chat interface
- **Visualizations**: Knowledge Graph, Job Workflows (React Flow)
- **Orchestration**: Background jobs (Redis + Temporal), Job Router
- **CLI**: init, scan, serve, query, sync

Reference snapshot: `.dev/reference/LightRAG-main` (version 1.4.9.11).

### What's Planned (Next Steps)

- Advanced Repository Graph features (Drill-down, Side panel - Phase 4 queued)
- Data Flow Visualizations (Lineage, Sync events - Phase 4 queued)
- Session memory (Phase 5)
- Cost tracking (Phase 5)
- Agent analytics (Phase 5)
- Git integration (Phase 5)
- Webhooks (Phase 5)

---

## Milestones Status

### M1: Scanner + Postgres Foundation (Completed)

**Delivered:** Scanner, Postgres Schema, Async Storage, Sync System.

### M2: Markdown Documentation (Completed)

**Delivered:** Jinja2 templates, Markdown Generator, Offline Mode.

### M3: RAG Query + LightRAG Index (Completed)

**Delivered:** LightRAG Client, Query Engine with Fallback, Semantic Search.

### M4: MCP Read + Report (Completed)

**Delivered:** MCP Server Core, Query Tools, Report Tools, CLI Agent integration.

### M5: Dashboard & Visuals (Completed)

**Delivered:** REST API, WebSocket Streams, Dashboard Scaffold, Chat, Settings, Basic Graphs.

---

## Fallback Mode

The system works in degraded mode when external dependencies are unavailable:

```
FULL MODE (all features):
  Scanner -> Parser -> Postgres -> LightRAG -> LLM -> Markdown -> Dashboard

OFFLINE MODE (no Postgres/LLM/vectors):
  Scanner -> Parser -> Markdown (basic)

  docrunch scan --offline

Use `--offline` to skip Postgres/LLM dependencies; metadata is cached locally under `.docrunch/offline_files.json`.
```

| Component       | Full Mode      | Offline Mode      |
| --------------- | -------------- | ----------------- |
| Scanner         | yes            | yes               |
| Parser          | yes            | yes               |
| Postgres        | yes            | no (skip)         |
| LightRAG        | yes            | no (skip)         |
| LLM Summaries   | yes            | no (templates)    |
| Markdown Output | yes (rich)     | yes (basic)       |
| Query           | yes (semantic) | yes (text search) |
| Dashboard       | yes            | no (api error)    |

---

## Dogfooding

Docrunch has been successfully run on itself ("dogfooding") to generate its own documentation and verify the integration.

---

## Post-MVP Phases (Current Roadmap)

| Phase   | Status      | Features                                                            |
| ------- | ----------- | ------------------------------------------------------------------- |
| Phase 2 | **Done**    | Intelligence (LLM providers, tree-sitter, CLI Bridge, Workflows)    |
| Phase 3 | **Done**    | MCP + Tasks (Server, Tools, Dashboard, Chat, Infrastructure)        |
| Phase 4 | **Ongoing** | Visualization (Repo Graph, Data Flow, Advanced React Flow features) |
| Phase 5 | Backlog     | Advanced Features (Memory, Analytics, Git, Cost, Webhooks)          |
| Phase 6 | Backlog     | Polish (Testing, Performance, Docs Cleanup)                         |

### Phase 4: Visualization Features (Active Queue)

React Flow-based interactive visualizations currently in the implementation queue:

| Feature                  | Description                                  | Task      |
| ------------------------ | -------------------------------------------- | --------- |
| **Repository Graph**     | Explore codebase structure and relationships | TASK-066+ |
| **Data Lineage**         | Visualize scan -> storage -> output flow     | TASK-088  |
| **Storage Health**       | Monitor Postgres, LightRAG, Redis, Temporal  | TASK-089  |
| **Sync Events**          | Real-time sync outbox visualization          | TASK-090  |
| **Entity Location Map**  | Track entities across storages               | TASK-091  |
| **Schema Visualization** | Postgres + LightRAG schema graphs            | TASK-092  |
| **Cross-Storage Diff**   | Identify out-of-sync entities                | TASK-094  |
| **Query Tracing**        | Trace queries across storage backends        | TASK-095  |

See [react_flow.md](./components/react_flow.md) and [DataFlowVisualization.md](./components/DataFlowVisualization.md) for details.
