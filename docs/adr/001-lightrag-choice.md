# ADR-001: LightRAG for Knowledge Graph

## Status

**Accepted**

## Context

Docrunch needs a vector database for semantic search. We also need relationship tracking between code components (functions, classes, imports).

## Options Considered

| Option          | Pros                                                            | Cons                            |
| --------------- | --------------------------------------------------------------- | ------------------------------- |
| **LightRAG**    | Built-in knowledge graph, semantic search, relationship queries | Newer library, less mature      |
| ChromaDB        | Simple, well-documented                                         | No built-in graph, just vectors |
| Pinecone        | Managed, scalable                                               | Cloud-only, cost, no graph      |
| Neo4j + Vectors | Mature graph DB                                                 | Two systems to manage           |

## Decision

Use **LightRAG** because:

1. Built-in knowledge graph aligns with our relationship tracking needs
2. Semantic search without external service
3. Single system for vectors + relationships
4. Local-first (no cloud dependency)

## Fallback Plan

If LightRAG proves problematic:

1. **Short-term**: Use Postgres for relationships + basic text search
2. **Long-term**: Migrate to ChromaDB + custom graph layer

## Consequences

- LightRAG is a hard dependency
- Need to handle document chunking carefully for large files
- Must monitor performance with large repositories
