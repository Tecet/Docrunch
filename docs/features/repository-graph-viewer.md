# Repository Graph Viewer (React Flow)

> **Status**: Proposed | **Phase**: 3 | **Priority**: Medium

Interactive visual exploration of scanned repositories using React Flow. Enables developers and AI agents to understand codebase structure, relationships, and navigate between connected components.

---

## Overview

The Repository Graph Viewer provides a visual representation of a scanned codebase as an interactive node-and-edge graph. Users can explore files, functions, classes, imports, database schemas, and React components, showing how they connect and drilling into subgraphs for detailed views.

### Core Approach: Hybrid Flat + Drill-down

-   **Default view**: Flat graph showing high-level components (folders, modules, services)
-   **Drill-down**: Click any node with children to open its subgraph in a focused view
-   **Breadcrumb navigation**: Track your path and jump back to any level
-   **Portal edges**: Cross-subgraph connections shown as dashed edges that navigate on click

---

## Entity Types (Node Types)

The viewer supports multiple entity types, with a **View Switcher** to focus on specific categories:

| Entity Type         | Icon          | Description                | Relationships                |
| ------------------- | ------------- | -------------------------- | ---------------------------- |
| **Folder**          | `[dir]`       | Directory node, expandable | Contains files/subfolders    |
| **File**            | `[file]`      | Source file                | Imports, exports             |
| **Function**        | `[fn]`        | Function definition        | Calls, called-by             |
| **Class**           | `[class]`     | Class definition           | Extends, implements, methods |
| **React Component** | `[component]` | JSX/TSX component          | Props, children, hooks       |
| **DB Table**        | `[db]`        | Database table/model       | Foreign keys, relations      |

### View Switcher Modes

-   **Structure View**: Files and folders only
-   **Dependencies View**: Focus on imports/exports
-   **Components View**: React components and their relationships
-   **Database View**: Tables, columns, and foreign keys
-   **Functions View**: Functions and call graph
-   **Full View**: All entities (may be dense for large repos)

---

## Node Interactions

Each node provides interactive tools accessible via icons or context menu:

### Primary Actions (Icon Bar)

| Icon     | Action               | Description                                              |
| -------- | -------------------- | -------------------------------------------------------- |
| `[docs]` | **View Docs**        | Open related Markdown documentation in side panel        |
| `[code]` | **View Code**        | Show code snippet in side panel with syntax highlighting |
| `[open]` | **Open in Editor**   | Open source file in VS Code / configured editor          |
| `[drill]` | **Drill Down**      | Enter subgraph for nodes with children                   |
| `[links]` | **Show Connections** | Highlight all connected nodes and edges                 |

### LLM Agent Actions (Submenu)

Users can select a node (or multiple nodes) and dispatch to LLM specialists:

| Action                   | Specialist          | Description                                    |
| ------------------------ | ------------------- | ---------------------------------------------- |
| **Code Sanity Check**    | QA Specialist       | Analyze code quality, suggest improvements     |
| **Security Audit**       | Security Specialist | Check for vulnerabilities, auth issues         |
| **Generate Docs**        | Librarian           | Create/update documentation for selected code  |
| **Generate Tests**       | QA Specialist       | Suggest test cases for selected function/class |

**Workflow**: Select node(s) -> Right-click -> "Send to LLM" -> Pick specialist -> View results in task system or side panel.

---

## Side Panel Features

A collapsible side panel provides detailed views:

### Code View

-   Syntax-highlighted code snippet
-   Line numbers and file path
-   "Open in Editor" button
-   Copy to clipboard

### Documentation View

-   Rendered Markdown from `docrunch-docs/`
-   Auto-generated summary if no docs exist
-   "Generate Docs" button for LLM generation

### Connections View

-   List of incoming and outgoing edges
-   Click to navigate to connected node
-   Filter by edge type (imports, calls, extends)

### Search Results

-   Fuzzy search across all nodes
-   Click result to focus and highlight node
-   Filter by entity type

---

## Navigation & UX

### Search & Jump

-   `Ctrl+K` / `Cmd+K` opens fuzzy search
-   Type to filter nodes by name, path, or type
-   Select result to pan and zoom to node

### Breadcrumb Trail

```
Repository > src > components > auth > LoginModal
```

-   Click any level to jump back
-   Shows current drill-down path

### Minimap

-   Overview of entire graph in corner
-   Current viewport highlighted
-   Click to navigate

### Zoom Controls

-   Mouse wheel zoom
-   Fit-to-screen button
-   Zoom to selection

---

## Backend Integration

### CLI Command

```bash
# Export graph for a repository
docrunch export-graph --output graph.json

# Export specific view
docrunch export-graph --view dependencies --output deps.json

# Export with layout positions (if previously saved)
docrunch export-graph --include-layout
```

### API Endpoints

| Endpoint                                    | Method | Description         |
| ------------------------------------------- | ------ | ------------------- |
| `/api/repos/:id/graph`                      | GET    | Full graph JSON     |
| `/api/repos/:id/graph/subgraph/:subgraphId` | GET    | Specific subgraph   |
| `/api/repos/:id/graph/layout`               | PUT    | Save node positions |
| `/api/repos/:id/graph/search?q=:query`      | GET    | Search nodes        |

### Data Model

```typescript
interface GraphNode {
    id: string;
    type:
        | "folder"
        | "file"
        | "function"
        | "class"
        | "component"
        | "db_table";
    label: string;
    path: string;
    meta: {
        language?: string;
        lines?: number;
        size?: number;
        signature?: string;
        columns?: Column[]; // for db_table
        props?: Prop[]; // for component
    };
    subgraphId?: string; // if drillable
    tools: NodeTool[];
    position?: { x: number; y: number }; // persisted layout
}

interface GraphEdge {
    id: string;
    source: string;
    target: string;
    type: "import" | "extends" | "implements" | "calls" | "fk" | "contains";
    label?: string;
    meta?: {
        symbols?: string[]; // imported symbols
    };
}

interface NodeTool {
    id: string;
    type: "doc" | "code" | "openEditor" | "llmAction";
    label: string;
    payload: string | LLMActionPayload;
}

interface LLMActionPayload {
    specialist: "qa" | "security" | "librarian";
    action:
        | "sanity_check"
        | "security_audit"
        | "generate_docs"
        | "generate_tests";
}
```

### Layout Persistence

-   Node positions saved per repository + view type
-   Stored in Postgres `graph_layouts` table
-   Layout versioned with scan timestamp
-   Auto-layout (dagre) used when no saved positions

---

## Technical Implementation

### Frontend Stack

-   **React Flow** (`reactflow` package) for graph rendering
-   **dagre** for automatic layout
-   **framer-motion** for animations
-   **Monaco Editor** for code snippets
-   **react-markdown** for docs rendering

### Performance Considerations

-   **Lazy loading**: Load subgraphs on demand
-   **Virtualization**: Only render visible nodes
-   **WebWorker**: Run layout calculations off main thread
-   **Chunking**: Collapse large folders into summary nodes
-   **Caching**: Cache layout positions and subgraph data

### Integration Points

-   **Scanner**: Emit graph data during scan
-   **Parser**: Provide AST-derived relationships
-   **Storage**: Graph data in Postgres, positions persisted
-   **MCP**: Expose graph queries for AI agents

---

## MVP vs Full Feature

### MVP Scope (Suggested)

For initial Phase 3 implementation, focus on:

1. **Basic file/folder graph** with import edges
2. **Single-level view** (no drill-down initially)
3. **View code in side panel**
4. **Search/jump to node**
5. **API endpoint** for graph data
6. **Auto-layout** (no position saving)

### Full Feature (Later)

Add iteratively:

-   Drill-down subgraphs with navigation
-   All entity types (DB, components, functions)
-   LLM agent dispatch from nodes
-   Layout persistence
-   View switcher
-   Advanced filters

---

## Relation to MVP Priority

### Why Phase 3--

The Repository Graph Viewer is **valuable but not essential** for the core Docrunch MVP:

| MVP Feature     | Required For | Graph Viewer Dependency    |
| --------------- | ------------ | -------------------------- |
| **Scanning**    | Core         | Graph consumes scan data   |
| **MCP Server**  | Core         | Graph could query MCP      |
| **Task System** | Core         | LLM actions create tasks   |
| **Dashboard**   | Core         | Graph is separate viewer   |
| **LightRAG**    | Core         | Graph could use for search |

**Phase 3 rationale**:

-   Phase 1: Foundation (scanning, storage, CLI) - **must complete first**
-   Phase 2: Intelligence (parsers, LLM, MCP) - **provides data for graph**
-   Phase 3: Visualization (graph viewer, advanced UI) - **builds on Phase 1+2**

### Dependencies

Before Graph Viewer can be fully functional:

-   File scanner must be working (TASK-004)
-   Tree-sitter parser for relationships (TASK-009)
-   Import/export detection in parser output
-   DB schema extraction
-   React component detection (JSX/TSX)

---

## Backlog Tasks

### TASK-066: Repository Graph Viewer - Core Infrastructure

-   **Type**: feature
-   **Priority**: medium
-   **Depends On**: [TASK-009, TASK-012]
-   **Status**: backlog

**Acceptance Criteria**:

-   [ ] `docrunch export-graph` CLI command
-   [ ] Graph JSON data model (GraphNode, GraphEdge)
-   [ ] API endpoint `/api/repos/:id/graph`
-   [ ] Basic React Flow viewer component (standalone page)
-   [ ] Auto-layout with dagre

---

### TASK-067: Repository Graph Viewer - Entity Types & Views

-   **Type**: feature
-   **Priority**: medium
-   **Depends On**: [TASK-066]
-   **Status**: backlog

**Acceptance Criteria**:

-   [ ] Node types: folder, file, function, class, component, db_table
-   [ ] Edge types: import, extends, implements, calls, fk, contains
-   [ ] View switcher UI component
-   [ ] View modes: Structure, Dependencies, Components, Database, Functions, Full
-   [ ] Custom node components for each entity type

---

### TASK-068: Repository Graph Viewer - Drill-down & Navigation

-   **Type**: feature
-   **Priority**: medium
-   **Depends On**: [TASK-067]
-   **Status**: backlog

**Acceptance Criteria**:

-   [ ] Subgraph drill-down on node click
-   [ ] Breadcrumb navigation component
-   [ ] Portal edges for cross-subgraph links
-   [ ] Animated transitions between subgraphs
-   [ ] Route-based navigation (shareable URLs)
-   [ ] Minimap for large graphs

---

### TASK-069: Repository Graph Viewer - Side Panel & Interactions

-   **Type**: feature
-   **Priority**: medium
-   **Depends On**: [TASK-067]
-   **Status**: backlog

**Acceptance Criteria**:

-   [ ] Code view in side panel (Monaco editor or syntax highlighter)
-   [ ] Docs view in side panel (rendered markdown)
-   [ ] Open in editor action (VSCode protocol link)
-   [ ] Fuzzy search with Ctrl+K / Cmd+K
-   [ ] Search endpoint: `GET /api/repos/:id/graph/search`
-   [ ] Show connections (highlight connected nodes on click)
-   [ ] Node icon bar: View Docs, View Code, Open Editor, Drill Down

---

### TASK-070: Repository Graph Viewer - LLM Agent Integration

-   **Type**: feature
-   **Priority**: low
-   **Depends On**: [TASK-069, TASK-012]
-   **Status**: backlog

**Acceptance Criteria**:

-   [ ] Context menu with LLM actions (right-click)
-   [ ] Multi-node selection
-   [ ] Dispatch to specialists: Code Sanity (QA), Security Audit, Generate Docs (Librarian), Generate Tests (QA)
-   [ ] Task creation from graph selection
-   [ ] Results displayed in side panel or task view
-   [ ] Progress indicator during LLM processing

---

### TASK-071: Repository Graph Viewer - Layout Persistence

-   **Type**: feature
-   **Priority**: low
-   **Depends On**: [TASK-066]
-   **Status**: backlog

**Acceptance Criteria**:

-   [ ] `graph_layouts` Postgres table
-   [ ] Save positions on node drag
-   [ ] Load saved positions on graph open
-   [ ] Per-view-type layout storage
-   [ ] Layout versioned with scan timestamp
-   [ ] API endpoint PUT `/api/repos/:id/graph/layout`

---

## Open Questions

1. **Standalone app or embedded--** Start as separate viewer, later integrate into dashboard as a route/tab--
2. **Export formats--** Support SVG/PNG export for documentation--
3. **Diff visualization--** Show changes between commits/branches--
4. **Multi-repo view--** Visualize dependencies across repositories--

---

## References

-   [React Flow Mindmap.md](./React%20Flow%20Mindmap.md) - Original drill-down concept
-   [React Flow to visualize a repository.md](./React%20Flow%20to%20visualize%20a%20repository.md) - Flat graph approach
-   [Workflow visualisations - React Flow.md](./Workflow%20visualisations%20-%20React%20Flow.md) - Runtime job visualization
