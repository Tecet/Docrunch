Use React Flow to visualize a repository scanned by docrunch. Below is a practical plan, data model, implementation notes, example code, and options for scaling and UX. Tell me which parts you want next (full PR, a minimal demo, or help adding export from docrunch).

1) High-level approach (summary)
- Let docrunch emit a graph-friendly JSON describing folders, files, import edges, DB schema objects (tables/models/relations), and any other entities (services, routes, tests).
- Build a React UI that consumes that JSON and renders it with React Flow (nodes + edges).
- Use an automatic layout (dagre or ELK) to position nodes; allow manual drag & save.
- Provide filters, collapsing/expanding subtrees, zoom-to-node, search, and tooltips with code previews / links to files.

2) Data model (suggested JSON)
- A single object with nodes[] and edges[].
- Node example:
  {
    id: "file:src/components/Button.tsx",
    type: "file",                  // file | folder | import | db_table | model | route | ...
    label: "Button.tsx",
    path: "src/components/Button.tsx",
    meta: { language: "ts", size: 144, lines: 120 },
    parent: "folder:src/components",
    collapsed: false,
    // optional UI hints
    color: "#61dafb",
    icon: "react"
  }
- Edge example:
  {
    id: "edge:1",
    source: "file:src/components/Button.tsx",
    target: "file:src/utils/format.ts",
    label: "import",
    meta: { importedSymbols: ["formatDate"] }
  }
- DB table node:
  {
    id: "db:users",
    type: "db_table",
    label: "users",
    meta: { columns: [{name:"id", type:"uuid"}, {name:"email",type:"string"}], pk:"id" }
  }
- This keeps the front-end generic and allows docrunch to add extra meta later.

3) How to get the data out of docrunch
- If docrunch already scans files, extend the scanner to emit the graph JSON (file tree + import edges + parsed DB schema).
- Implementation tips:
  - Folder tree: straightforward from filesystem traversal.
  - Imports: parse AST per language (esbuild / ts-morph for JS/TS; Python ast or jedi; go/parser; rust-analyzer for Rust). If docrunch already has parsers, add a step to record import edges.
  - DB schemas: for ORM-based projects (Prisma, TypeORM, Django models) parse model definitions; for raw SQL parse CREATE TABLE statements.
- Output a single .json per repo + optional streaming updates for large repos.

4) React Flow implementation (concept + minimal example)
- Dependencies: react-flow-renderer (or reactflow package), dagre (for layout), react, axios/fetch.
- Core ideas:
  - Map docrunch nodes -> React Flow nodes (id, type, position, data).
  - Map edges -> React Flow edges.
  - Run dagre layout to compute positions before rendering.
  - Provide custom node types for folder, file, db_table (to display columns).
  - Add Controls, MiniMap, Background, and a side panel for details.

Minimal React pseudocode (conceptual)
- transform graph JSON into nodes/edges
- run dagre layout
- use React Flow with onNodeDragStop to let user reposition and save

Notes:
- Use node.data to include path and meta for tooltips and click actions (open file in repo viewer).
- Use grouping: make folder nodes act as "parent" and use React Flow's nested nodes / node types to visually collapse.

5) Layout and UX details
- For folder tree: allow an alternate tree-layout mode (vertical tree) for folder/file view.
- For import relations: use graph view (directed edges). Use edge bundling or curved edges to reduce clutter.
- DB schemas: render tables with columns inside node; allow clicking a column to show references/foreign keys.
- Filtering: by file type, language, directory depth, import count, or by entity type (show only db/schema/imports).
- Progressive loading: start with top-level folders; expand on demand to avoid rendering thousands of nodes at once.
- Clustering: collapse folders into a single node with a badge count; expand on click.

6) Performance & scaling
- Large repo (thousands of nodes): do server-side layout and progressive loading.
- Virtualize nodes shown (only render visible part) or use collapsing to reduce active node count.
- Use WebWorker for layout/graph processing (dagre can block main thread).
- Cache layout positions per repo/branch to avoid recomputation.

7) Example server-side transform (Node.js pseudocode)
- Read docrunch scan results -> build nodes/edges -> write nodes_edges.json
- Provide an endpoint GET /api/repos/:id/graph that returns JSON

8) Visual styles and node types
- Folder node: folder icon, click to expand; background color per folder depth.
- File node: icon by language; small stats (imports count, lines).
- DB table node: show column list; highlight FK relationships with colored edges.
- Import edge styling: arrow, label shows symbols or counts; color by type (devDeps vs deps).
- Use legends and color palette for quick reading.

9) Example features to add
- Click a node -> side panel shows file contents, AST snippet, metrics.
- Click an import edge -> shows imported symbols or call graph.
- Search / fuzzy jump to node.
- Export view as SVG or JSON.
- Link nodes to GitHub blob URLs (open file in GitHub).
- Timeline: show changes across commits (diff visualization).

10) Implementation estimate (rough)
- Minimal demo (show tree + imports) with docrunch JSON -> React Flow viewer: 1–2 days.
- Full integration (DB schema parsing, collapse/expand, search, save layout): 1–2 weeks.
- Production-ready with performance optimizations and multi-language parsers: ~3–6 weeks depending on scope.

11) Concrete next steps I can do for you
- Help design the JSON schema and add an export to docrunch; or
- Add a React Flow viewer component and example UI; or
- Open a PR in this repo that adds a small demo viewer and endpoint that serves sample graph JSON.
Which do you want me to do next? If PR, confirm repo (you are owner) and whether to target main, and whether you want a minimal demo or a fuller feature set. Also tell me:
- Which languages you care most about (JS/TS/Python/Go/...).
- Whether docrunch already produces AST/import metadata or I should add parsers.
- Do you prefer client-side scanning (upload repo to browser) or server-side (docrunch server produces JSON)?

If you want, I can draft a starter PR that:
- Adds a command docrunch export-graph -> graph.json
- Adds a small React app under /web/visualizer that loads graph.json and renders via React Flow
Answer which option and I’ll prepare the implementation details or the PR.
