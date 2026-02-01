# React Flow Visualizers

> Component documentation for React Flow-based visualization features in Docrunch.

---

## Overview

Docrunch uses [React Flow](https://reactflow.dev/) for interactive graph-based visualizations. Four distinct visualizers are planned:

| Visualizer              | Purpose                                        | Page Route             |
| ----------------------- | ---------------------------------------------- | ---------------------- |
| **Repository Graph**    | Explore codebase structure and relationships   | `/graph/:repoId`       |
| **Workflow Visualizer** | Monitor Temporal/Redis job status in real-time | `/dashboard/workflows` |
| **Knowledge Graph**     | Navigate LightRAG entities and relationships   | `/dashboard/knowledge` |
| **Data Flow**           | Monitor storage, sync, and query flows         | `/dashboard/dataflow`  |

---

## Repository Graph Viewer

The primary visualization tool for exploring scanned repositories.

### Features

-   **Hybrid navigation**: Flat overview + drill-down into subgraphs
-   **Multiple entity types**: Files, functions, classes, components, DB tables
-   **View switcher**: Focus on Structure, Dependencies, Components, Database, or Functions
-   **Interactive nodes**: View code, view docs, open in editor, dispatch to LLM
-   **Search**: Fuzzy jump to any node with `Ctrl+K`
-   **Layout persistence**: Drag nodes and save positions

### Node Types

| Type        | Icon          | Description           |
| ----------- | ------------- | --------------------- |
| `folder`    | `[dir]`       | Directory, expandable |
| `file`      | `[file]`      | Source file           |
| `function`  | `[fn]`        | Function definition   |
| `class`     | `[class]`     | Class definition      |
| `component` | `[component]` | React/JSX component   |
| `db_table`  | `[db]`        | Database table/model  |

### Edge Types

| Type         | Style       | Description              |
| ------------ | ----------- | ------------------------ |
| `import`     | Solid arrow | File imports another     |
| `extends`    | Dashed      | Class inheritance        |
| `implements` | Dotted      | Interface implementation |
| `calls`      | Thin arrow  | Function call            |
| `fk`         | Bold arrow  | Foreign key reference    |
| `contains`   | Grouped     | Parent-child containment |

### API Endpoints

```
GET  /api/repos/:id/graph              # Full graph JSON
GET  /api/repos/:id/graph/subgraph/:subgraphId   # Subgraph data
PUT  /api/repos/:id/graph/layout       # Save node positions
GET  /api/repos/:id/graph/search?q=    # Search nodes
```

### CLI Command

```bash
# Export graph for a repository
docrunch export-graph --output graph.json

# Export specific view
docrunch export-graph --view dependencies

# Include saved layout positions
docrunch export-graph --include-layout
```

### Side Panel

A collapsible panel provides detailed views:

-   **Code View**: Syntax-highlighted source with line numbers
-   **Docs View**: Rendered markdown from `docrunch-docs/`
-   **Connections View**: List of incoming/outgoing edges

### LLM Integration

Right-click any node to dispatch to LLM specialists:

-   **Code Sanity Check** -> QA Specialist
-   **Security Audit** -> Security Specialist
-   **Generate Docs** -> Librarian Specialist
-   **Generate Tests** -> QA Specialist

Results appear in the side panel or as new tasks.

---

## Workflow Visualizer

Real-time monitoring of background jobs and Temporal workflows.

### Features

-   **Live updates**: WebSocket connection for instant status changes
-   **Node states**: Running (blue/spinner), Completed (green), Failed (red)
-   **Compensation paths**: Rollback logic shown as dashed red edges
-   **Drill-down**: Expand workflow to see individual activities

### Node Types

| Type        | Description        |
| ----------- | ------------------ |
| `workflow`  | Temporal workflow  |
| `activity`  | Temporal activity  |
| `redis_job` | Dramatiq/Redis job |

### WebSocket Events

```
WS /ws/workflows
```

Events: `job_started`, `job_completed`, `job_failed`, `workflow_updated`

---

## Knowledge Graph Visualizer

Explore the LightRAG semantic graph of entities and relationships.

### Features

-   **Entity types**: Document, Code File, Concept, Pattern
-   **Relationship navigation**: Click edges to traverse
-   **Search/filter**: Find nodes by type or name
-   **Scan updates**: WebSocket refresh after new scans

---

## Technical Stack

| Component       | Library                | Purpose               |
| --------------- | ---------------------- | --------------------- |
| Graph rendering | `reactflow`            | Nodes, edges, canvas  |
| Layout          | `dagre`                | Automatic positioning |
| Animations      | `framer-motion`        | Transitions           |
| Code display    | `@monaco-editor/react` | Syntax highlighting   |
| Markdown        | `react-markdown`       | Docs rendering        |

### Performance

-   **Lazy loading**: Subgraphs loaded on demand
-   **Virtualization**: Only visible nodes rendered
-   **WebWorker**: Layout calculations off main thread
-   **Caching**: Layout positions cached per view

---

## Data Model

```typescript
interface GraphNode {
    id: string;
    type: "folder" | "file" | "function" | "class" | "component" | "db_table";
    label: string;
    path: string;
    meta: Record<string, unknown>;
    subgraphId?: string;
    position?: { x: number; y: number };
}

interface GraphEdge {
    id: string;
    source: string;
    target: string;
    type: "import" | "extends" | "implements" | "calls" | "fk" | "contains";
    label?: string;
}
```

---

## Related Tasks

| Task     | Description                           | Status  |
| -------- | ------------------------------------- | ------- |
| TASK-064 | Workflow Visualizer                   | backlog |
| TASK-065 | Knowledge Graph Visualizer            | backlog |
| TASK-066 | Repository Graph - Core               | backlog |
| TASK-067 | Repository Graph - Entity Types       | backlog |
| TASK-068 | Repository Graph - Drill-down         | backlog |
| TASK-069 | Repository Graph - Side Panel         | backlog |
| TASK-070 | Repository Graph - LLM Integration    | backlog |
| TASK-071 | Repository Graph - Layout Persistence | backlog |
| TASK-096 | Graph Performance Optimization        | backlog |
| TASK-097 | Graph Accessibility (a11y)            | backlog |
| TASK-098 | Graph Caching Strategy                | backlog |
| TASK-099 | Graph Security & Permissions          | backlog |

Data flow visualization tasks are tracked in TASK-088 through TASK-095 (see Data Flow Visualization).

---

## References

-   [Feature Spec: Repository Graph Viewer](../features/repository-graph-viewer.md)
-   [React Flow Advanced Features](./react_flow_features.md)
-   [Data Flow Visualization](./DataFlowVisualization.md)
-   [Development Guide](../development/react-flow-guide.md)
-   [React Flow Documentation](https://reactflow.dev/)
-   [dagre Layout Library](https://github.com/dagrejs/dagre)
