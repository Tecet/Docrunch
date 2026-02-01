# React Flow Advanced Features

> Future enhancements for React Flow visualizers beyond core functionality.

---

## Overview

This document outlines advanced features planned for the React Flow visualization system. These build on the core [Repository Graph Viewer](./react_flow.md) and extend it with intelligent analysis, collaboration, and integration capabilities.

---

## Analysis Features

### Diff Visualization (TASK-072)

Visualize code changes between commits, branches, or PRs:

-   **Node coloring**: Added (green), Modified (orange), Deleted (red)
-   **Toggle view**: Switch between "current state" and "diff mode"
-   **Side panel**: Click node to see file diff with syntax highlighting
-   **Use case**: "What did this PR actually touch?"

```
GET /api/repos/:id/graph/diff?from=main&to=feature-branch
```

---

### Impact Analysis (TASK-073)

"What will break if I change this?"

-   **Downstream highlighting**: Select node -> show all dependents
-   **Severity colors**: Direct (red), Indirect (orange), Distant (yellow)
-   **Export report**: Generate impact assessment document
-   **Use case**: Safe refactoring, understanding blast radius

```
GET /api/repos/:id/graph/impact/:nodeId
```

---

### Code Coverage Overlay (TASK-074)

Visualize test coverage on the graph:

-   **Coverage coloring**: >80% (green), 40-80% (yellow), <40% (red)
-   **Test edges**: Show which test files cover each function
-   **Tooltip**: Coverage percentage on hover
-   **Use case**: "Where should I add tests?"

Requires integration with coverage tools (pytest-cov, istanbul, etc.)

---

### Complexity Heatmap (TASK-075)

Show code complexity visually:

-   **Node size**: Based on lines of code or cyclomatic complexity
-   **Color gradient**: Green (simple) to red (complex)
-   **Hotspot detection**: Highlight maintainability issues
-   **Use case**: Tech debt visualization, refactoring targets

---

## Collaboration Features

### Annotations & Bookmarks (TASK-076)

Let users mark up the graph:

-   **Pin notes**: Add comments to nodes ("needs refactoring", "security concern")
-   **Bookmarks**: Save named paths through the graph
-   **Sharing**: Share annotations with team members
-   **Persistence**: Store in database per user/team
-   **Use case**: Code review, knowledge sharing, onboarding

---

### Collaborative Cursors (TASK-077)

Real-time multi-user exploration (like Figma):

-   **Live cursors**: See where teammates are looking
-   **Follow mode**: Jump to another user's view
-   **WebSocket sync**: Real-time position updates
-   **Use case**: Remote pair programming, architecture reviews

---

### Guided Tours (TASK-078)

Scripted walkthroughs of the codebase:

-   **Tour creation**: Define sequence of nodes with narration
-   **Playback**: Auto-pilot mode that pans through tour
-   **Controls**: Play, pause, skip, speed adjustment
-   **Templates**: "Auth flow", "Data pipeline", "New contributor intro"
-   **Use case**: Onboarding, architecture documentation

---

## AI-Powered Features

### Auto-Clustering (TASK-079)

AI-driven grouping of related nodes:

-   **LLM analysis**: Understand logical groupings from code semantics
-   **Virtual containers**: Group files into modules/features
-   **Suggestions**: "These 5 files form the auth module"
-   **One-click apply**: Create subgraph from suggestion
-   **Use case**: Understanding unfamiliar codebases

---

### Natural Language Queries (TASK-080)

Ask questions in plain English:

-   **Query bar**: "Show me everything related to user authentication"
-   **Graph highlighting**: Matches appear highlighted
-   **Follow-up**: "Now show what calls the payment API"
-   **Voice input**: Optional speech-to-text
-   **Use case**: Exploration without knowing code structure

```
POST /api/repos/:id/graph/query
{ "question": "What calls the payment API?" }
```

---

### Architecture Suggestions (TASK-081)

AI-driven code health recommendations:

-   **Dependency analysis**: "This module has too many dependencies"
-   **Coupling detection**: "These files change together, consider co-location"
-   **Circular dependencies**: "Detected cycle between X and Y"
-   **Improvement tasks**: One-click create refactoring task
-   **Use case**: Continuous architecture health monitoring

---

## Export & Integration

### Export to Diagram Tools (TASK-082)

Export graphs to external formats:

-   **Mermaid**: For markdown documentation
-   **Draw.io/Lucidchart**: For presentations (.drawio, .xml)
-   **PlantUML**: For ADRs and technical specs
-   **SVG/PNG**: Static image export
-   **Use case**: Architecture documentation, presentations

```
GET /api/repos/:id/graph/export?format=mermaid
GET /api/repos/:id/graph/export?format=svg
```

---

### Embed in Markdown (TASK-083)

Auto-generate and embed graph snapshots in docs:

-   **Static SVG**: Generate graph as image
-   **Auto-update**: Re-generate when code changes
-   **Embed syntax**: `![graph](./graphs/auth-module.svg)`
-   **Output path**: `docrunch-docs/graphs/` (uses `storage.docs_output`)
-   **Target files**: README.md, architecture docs
-   **Use case**: Keep documentation in sync with code

---

### IDE Integration (TASK-084)

VSCode extension with graph panel:

-   **Mini graph**: Sidebar showing current file's connections
-   **Bidirectional navigation**: Click node -> open file, navigate file -> highlight node
-   **Live updates**: Graph refreshes as you navigate
-   **Quick actions**: Right-click for LLM actions
-   **Use case**: Live navigation aid during development

---

## Advanced Use Cases

### Multi-Repository View (TASK-085)

Visualize dependencies across multiple repositories:

-   **Cross-repo edges**: Show npm/pip dependencies between repos
-   **Microservice map**: Service-to-service communication
-   **Shared libraries**: Highlight common dependencies
-   **Impact across repos**: "Changing this affects 3 other repos"
-   **Use case**: Monorepo or microservices architecture

---

### Time-lapse View (TASK-086)

Animate codebase evolution over time:

-   **Commit playback**: Watch architecture emerge across commits
-   **Speed controls**: 1x, 2x, 5x, 10x playback
-   **Milestone markers**: Tag significant commits
-   **Regression detection**: "Complexity increased 40% in Q3"
-   **Use case**: Understanding project history, retrospectives

---

### Custom Node Templates (TASK-087)

User-defined node types for domain-specific visualization:

-   **Template editor**: Define custom node appearance and data
-   **Domain types**: "API endpoint", "GraphQL resolver", "Terraform resource"
-   **Import/export**: Share templates with team
-   **Auto-detection**: Infer type from file patterns
-   **Use case**: Specialized visualization for specific tech stacks

---

## Implementation Priority

| Phase  | Tasks   | Focus Area                                        |
| ------ | ------- | ------------------------------------------------- |
| **3A** | 072-075 | Analysis (Diff, Impact, Coverage, Complexity)     |
| **3B** | 076-078 | Collaboration (Annotations, Cursors, Tours)       |
| **3C** | 079-081 | AI Features (Clustering, NL Queries, Suggestions) |
| **3D** | 082-084 | Export & Integration                              |
| **3E** | 085-087 | Advanced (Multi-repo, Time-lapse, Templates)      |

---

## Related Documentation

-   [React Flow Core](./react_flow.md) - Base visualizer documentation
-   [Repository Graph Viewer Spec](../features/repository-graph-viewer.md) - Full feature specification
-   [BACKLOG](../../.dev/tasks/BACKLOG.md) - Task definitions
