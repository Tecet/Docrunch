# React Flow Development Guide

> Guidelines for developing and extending React Flow visualizers in Docrunch.

---

## Overview

Docrunch uses [React Flow](https://reactflow.dev/) for interactive graph-based visualizations. This guide covers custom nodes, visualizer pages, WebSocket events, and graph data contracts.

---

## Project Structure

```
web/
  src/
    pages/
      RepositoryGraphViewer.tsx    # Main graph viewer
      WorkflowVisualizer.tsx       # Temporal/Redis jobs
      KnowledgeGraphVisualizer.tsx # LightRAG entities
      DataFlowVisualizer.tsx       # Storage/sync monitoring
    components/
      graph/
        nodes/                     # Custom node types
          FolderNode.tsx
          FileNode.tsx
          FunctionNode.tsx
          ClassNode.tsx
          ComponentNode.tsx
          DbTableNode.tsx
          index.ts                 # Node type registry
        edges/                     # Custom edge types
          ImportEdge.tsx
          PortalEdge.tsx
          SyncEdge.tsx
          index.ts
        controls/                  # UI controls
          ViewSwitcher.tsx
          SearchModal.tsx
          Breadcrumbs.tsx
          RefreshButton.tsx
          LiveToggle.tsx
        panels/                    # Side panels
          SidePanel.tsx
          CodeView.tsx
          DocsView.tsx
          ConnectionsView.tsx
      dataflow/
        DataFlowCanvas.tsx         # Base React Flow canvas
        StorageNode.tsx            # Storage nodes
        ProcessorNode.tsx          # Worker/router nodes
        DataEdge.tsx               # Animated data flow edges
        TracePanel.tsx             # Query trace panel
    hooks/
      useGraphData.ts              # Fetch and transform graph data
      useGraphLayout.ts            # Layout persistence
      useGraphWebSocket.ts         # Live updates
      useGraphSearch.ts            # Fuzzy search
    utils/
      graphTransform.ts            # API -> React Flow format
      layoutEngine.ts              # Dagre wrapper
      graphConstants.ts            # Colors, sizes, icons
```

---

## Creating Custom Node Types

### 1. Define the Node Component

```tsx
// web/src/components/graph/nodes/FunctionNode.tsx
import { memo } from "react";
import { Handle, Position, NodeProps } from "reactflow";

interface FunctionNodeData {
    label: string;
    signature: string;
    path: string;
    lines: number;
}

function FunctionNode({ data, selected }: NodeProps<FunctionNodeData>) {
    return (
        <div
            className={`graph-node function-node ${selected ? "selected" : ""}`}
        >
            <Handle type="target" position={Position.Top} />

            <div className="node-header">
                <span className="node-icon">[fn]</span>
                <span className="node-label">{data.label}</span>
            </div>

            <div className="node-details">
                <code>{data.signature}</code>
                <span className="node-meta">{data.lines} lines</span>
            </div>

            <Handle type="source" position={Position.Bottom} />
        </div>
    );
}

export default memo(FunctionNode);
```

### 2. Register the Node Type

```tsx
// web/src/components/graph/nodes/index.ts
import FolderNode from "./FolderNode";
import FileNode from "./FileNode";
import FunctionNode from "./FunctionNode";
import ClassNode from "./ClassNode";
import ComponentNode from "./ComponentNode";
import DbTableNode from "./DbTableNode";

export const nodeTypes = {
    folder: FolderNode,
    file: FileNode,
    function: FunctionNode,
    class: ClassNode,
    component: ComponentNode,
    db_table: DbTableNode,
};
```

Data flow visualizers register their node types in `web/src/components/dataflow` to keep storage/processor nodes separate.

### 3. Use in React Flow

```tsx
import ReactFlow from "reactflow";
import { nodeTypes } from "./components/graph/nodes";

function GraphViewer({ nodes, edges }) {
    return (
        <ReactFlow nodes={nodes} edges={edges} nodeTypes={nodeTypes} fitView />
    );
}
```

---

## Adding a New Visualizer Page

### 1. Create the Page Component

```tsx
// web/src/pages/MyNewVisualizer.tsx
import { useCallback } from "react";
import ReactFlow, {
    Controls,
    MiniMap,
    Background,
    useNodesState,
    useEdgesState,
} from "reactflow";
import { nodeTypes } from "../components/graph/nodes";
import { useGraphData } from "../hooks/useGraphData";
import SidePanel from "../components/graph/panels/SidePanel";

export default function MyNewVisualizer() {
    const { data, isLoading, error } = useGraphData("/api/my-endpoint");
    const [nodes, setNodes, onNodesChange] = useNodesState(data?.nodes || []);
    const [edges, setEdges, onEdgesChange] = useEdgesState(data?.edges || []);

    const onNodeClick = useCallback((event, node) => {
        // Handle node click
    }, []);

    if (isLoading) return <div>Loading...</div>;
    if (error) return <div>Error: {error.message}</div>;

    return (
        <div className="visualizer-container">
            <ReactFlow
                nodes={nodes}
                edges={edges}
                onNodesChange={onNodesChange}
                onEdgesChange={onEdgesChange}
                onNodeClick={onNodeClick}
                nodeTypes={nodeTypes}
                fitView
            >
                <Controls />
                <MiniMap />
                <Background />
            </ReactFlow>
            <SidePanel />
        </div>
    );
}
```

### 2. Add Route

```tsx
// web/src/App.tsx
import MyNewVisualizer from "./pages/MyNewVisualizer";

<Route path="/visualizer/my-new" element={<MyNewVisualizer />} />;
```

---

## WebSocket Event Protocol

### Connecting to Live Updates

```tsx
// web/src/hooks/useGraphWebSocket.ts
import { useEffect, useState } from "react";

type GraphEventType =
    | "node_added"
    | "node_updated"
    | "node_removed"
    | "edge_added"
    | "edge_removed"
    | "sync_event"
    | "health_update"
    | "lineage_event"
    | "workflow_updated"
    | "query_trace";

interface GraphEvent {
    type: GraphEventType;
    payload: unknown;
    timestamp: string;
}

export function useGraphWebSocket(endpoint: string) {
    const [events, setEvents] = useState<GraphEvent[]>([]);
    const [isConnected, setIsConnected] = useState(false);
    const wsBaseUrl = window.location.origin.replace("http", "ws");

    useEffect(() => {
        const ws = new WebSocket(`${wsBaseUrl}${endpoint}`);

        ws.onopen = () => setIsConnected(true);
        ws.onclose = () => setIsConnected(false);

        ws.onmessage = (event) => {
            const data: GraphEvent = JSON.parse(event.data);
            setEvents((prev) => [...prev, data]);
        };

        return () => ws.close();
    }, [endpoint, wsBaseUrl]);

    return { events, isConnected };
}
```

### Event Types

| Event             | Payload                  | Description                  |
| ----------------- | ------------------------ | ---------------------------- |
| `node_added`      | `{ node: GraphNode }`    | New node added to graph      |
| `node_updated`    | `{ id, changes }`        | Node properties changed      |
| `node_removed`    | `{ id }`                 | Node removed from graph      |
| `edge_added`      | `{ edge: GraphEdge }`    | New edge added               |
| `edge_removed`    | `{ id }`                 | Edge removed                 |
| `sync_event`      | `{ event: SyncEvent }`   | Sync event status change     |
| `health_update`   | `{ backend, status }`    | Storage health change        |
| `lineage_event`   | `{ event }`              | Lineage event update         |
| `workflow_updated` | `{ workflow_id, status }` | Workflow execution update   |
| `query_trace`     | `{ trace: QueryTrace }`  | Query trace update           |

---

## Graph JSON Schema

### Backend Response Format

```typescript
interface GraphResponse {
    nodes: GraphNode[];
    edges: GraphEdge[];
    meta: {
        total_nodes: number;
        total_edges: number;
        generated_at: string;
        scan_id: string;
    };
}

type GraphNodeType =
    | "folder"
    | "file"
    | "function"
    | "class"
    | "component"
    | "db_table"
    | "workflow"
    | "activity"
    | "redis_job"
    | "source"
    | "storage"
    | "processor"
    | "output";

interface GraphNode {
    id: string;
    type: GraphNodeType;
    label: string;
    path: string;
    meta: Record<string, unknown>;
    subgraphId?: string;
    position?: { x: number; y: number };
}

type GraphEdgeType =
    | "import"
    | "extends"
    | "implements"
    | "calls"
    | "fk"
    | "contains"
    | "read"
    | "write"
    | "sync";

interface GraphEdge {
    id: string;
    source: string;
    target: string;
    type: GraphEdgeType;
    label?: string;
    animated?: boolean;
}
```

### Transform to React Flow Format

```typescript
// web/src/utils/graphTransform.ts
import { Node, Edge } from "reactflow";

export function transformToReactFlow(response: GraphResponse): {
    nodes: Node[];
    edges: Edge[];
} {
    const nodes: Node[] = response.nodes.map((node) => ({
        id: node.id,
        type: node.type,
        position: node.position || { x: 0, y: 0 },
        data: {
            label: node.label,
            path: node.path,
            ...node.meta,
        },
    }));

    const edges: Edge[] = response.edges.map((edge) => ({
        id: edge.id,
        source: edge.source,
        target: edge.target,
        type: edge.type,
        label: edge.label,
        animated: edge.animated,
    }));

    return { nodes, edges };
}
```

---

## Layout with Dagre

```typescript
// web/src/utils/layoutEngine.ts
import dagre from "dagre";
import { Node, Edge } from "reactflow";

const NODE_WIDTH = 172;
const NODE_HEIGHT = 36;

export function applyDagreLayout(
    nodes: Node[],
    edges: Edge[],
    direction: "TB" | "LR" = "TB"
): Node[] {
    const g = new dagre.graphlib.Graph();
    g.setDefaultEdgeLabel(() => ({}));
    g.setGraph({ rankdir: direction });

    nodes.forEach((node) => {
        g.setNode(node.id, { width: NODE_WIDTH, height: NODE_HEIGHT });
    });

    edges.forEach((edge) => {
        g.setEdge(edge.source, edge.target);
    });

    dagre.layout(g);

    return nodes.map((node) => {
        const nodeWithPosition = g.node(node.id);
        return {
            ...node,
            position: {
                x: nodeWithPosition.x - NODE_WIDTH / 2,
                y: nodeWithPosition.y - NODE_HEIGHT / 2,
            },
        };
    });
}
```

---

## Styling Constants

```typescript
// web/src/utils/graphConstants.ts

export const NODE_COLORS = {
    folder: "#FFD700",
    file: "#87CEEB",
    function: "#90EE90",
    class: "#DDA0DD",
    component: "#61dafb",
    db_table: "#FFA07A",
};

export const EDGE_STYLES = {
    import: { stroke: "#888", strokeWidth: 1 },
    extends: { stroke: "#9370DB", strokeWidth: 2, strokeDasharray: "5,5" },
    calls: { stroke: "#32CD32", strokeWidth: 1 },
    fk: { stroke: "#FF6347", strokeWidth: 2 },
};

export const NODE_ICONS = {
    folder: "[dir]",
    file: "[file]",
    function: "[fn]",
    class: "[class]",
    component: "[component]",
    db_table: "[db]",
};
```

---

## Data Freshness Strategy

Graphs use **manual refresh + auto-refresh toggle**:

```tsx
// In graph viewer component
const [autoRefresh, setAutoRefresh] = useState(false);

// Manual refresh button
<button onClick={() => refetch()}>
  Refresh
</button>

// Auto-refresh toggle
<label>
  <input
    type="checkbox"
    checked={autoRefresh}
    onChange={(e) => setAutoRefresh(e.target.checked)}
  />
  Auto-refresh
</label>

// Auto-refresh effect
useEffect(() => {
  if (!autoRefresh) return;
  const interval = setInterval(refetch, 30000); // 30s
  return () => clearInterval(interval);
}, [autoRefresh, refetch]);
```

---

## Testing

### Unit Tests (Jest)

```typescript
import { transformToReactFlow } from "@/utils/graphTransform";

describe("transformToReactFlow", () => {
    it("maps nodes and edges into React Flow format", () => {
        const response = {
            nodes: [{ id: "n1", type: "file", label: "a.py", path: "a.py", meta: {} }],
            edges: [{ id: "e1", source: "n1", target: "n1", type: "import" }],
            meta: { total_nodes: 1, total_edges: 1, generated_at: "", scan_id: "" },
        };
        const result = transformToReactFlow(response);
        expect(result.nodes).toHaveLength(1);
        expect(result.edges).toHaveLength(1);
    });
});
```

### E2E Tests (Playwright)

```typescript
// Test graph interactions
test("clicking a node opens the side panel", async ({ page }) => {
    await page.goto("/graph/repo-1");
    await page.click('[data-testid=\"node-auth-login\"]');
    await expect(page.locator(\".side-panel\")).toBeVisible();
});
```

---

## Security Checklist

Before deploying graph features:

-   [ ] Rate limiting on all graph and lineage endpoints
-   [ ] Sanitize node labels to prevent XSS
-   [ ] Validate graph search query input
-   [ ] Authenticate WebSocket connections
-   [ ] Repository-level permissions for graph access
-   [ ] Check permissions before LLM dispatch
-   [ ] Audit log LLM actions from graph
-   [ ] CSRF tokens on mutation endpoints

---

## Related Documentation

-   [React Flow Core](../components/react_flow.md)
-   [React Flow Advanced Features](../components/react_flow_features.md)
-   [Data Flow Visualization](../components/DataFlowVisualization.md)
-   [Interfaces (API contracts)](../interfaces.md)
