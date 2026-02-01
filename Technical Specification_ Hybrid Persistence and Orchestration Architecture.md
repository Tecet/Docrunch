### Technical Specification: Hybrid Persistence and Orchestration Architecture

#### 1\. Architectural Design Philosophy and System Context

In the landscape of repository intelligence, Docrunch represents a strategic synthesis between static code analysis and semantic AI reasoning. The architecture is designed to bridge the gap between deterministic Abstract Syntax Trees (AST)—extracted via  **tree-sitter** —and the fluid, relational understanding required for agentic AI. This hybrid approach ensures that the "context loss" typically associated with stateless LLM interactions is mitigated by a persistent, evolving knowledge base.The "Docrunch approach" differentiates itself by rejecting the limitations of a single-store solution. Instead, it employs a multi-model storage strategy—Relational (Postgres), Vector/Graph (LightRAG), and File-based (Markdown)—to create a resilient system where data availability is decoupled from individual service health. By using Postgres as a structural anchor, LightRAG for semantic connections, and Markdown for portable documentation, the system maintains reliability and enables graceful degradation, ensuring the AI remains grounded in the actual source code.

##### Technology Stack Summary

Component,Technology,Purpose  
Relational Database,Postgres 16,"Authoritative source of truth for metadata, task states, and settings"  
Parsing / AST,tree-sitter,"High-speed, polyglot structural analysis of source code"  
Vector / Graph Store,LightRAG (v1.4.9.11),Semantic search and knowledge graph relationship mapping  
Task Queue (Fast),Redis / Dramatiq,Background job processing for low-latency tasks (\<30s)  
Workflow Engine,Temporal,"Durable orchestration for long-running, multi-step workflows"  
Documentation,Markdown / Jinja2,File-based output for human-readable and portable repository docs  
These components form the backbone of the "Source of Truth" model, where structured metadata is systematically transformed into semantic intelligence.

#### 2\. The Primary Persistence Layer: Postgres as the Source of Truth

Postgres 16 serves as the authoritative source of truth for all metadata and task states within Docrunch. Its selection as the primary persistence layer is driven by the need for strict relational integrity and the ability to manage complex state transitions across concurrent repository scans. While specialized stores handle vectors and files, Postgres provides the structural validation required for a professional-grade intelligence platform.

##### Database Responsibilities

* **Structured Metadata:**  Managing files, scan\_runs, and relationships (extracted via tree-sitter).  
* **Task Management:**  Tracking the lifecycle of agentic operations, including task\_history, completion\_reports, findings, and decisions.  
* **LLM Settings:**  Hosting llm\_providers and llm\_profiles which define the behavioral constraints for AI specialists.  
* **Synchronization Outbox:**  Driving the consistency engine via sync\_events and storage\_sync tables.

##### Schema Architecture and Security Nuance

The schema is partitioned into functional domains to ensure scalability. Core Metadata tracks the physical and logical structure of the repository, while LLM Settings manage the specialist workforce. For security, all provider credentials and API keys are encrypted at rest using the DOCRUNCH\_KMS\_KEY. Critically, while the database holds the encrypted secrets, the Dashboard UI is responsible for  **masking**  these credentials to prevent exposure during administrative sessions.

##### Operational Reliability

By utilizing  **SQLAlchemy 2.0 async**  with the  **asyncpg**  driver and  **Alembic**  migrations, Docrunch achieves high-concurrency non-blocking I/O. This is essential for preventing the scanner from stalling during high-latency LLM calls. The combination of asynchronous operations and the Outbox Pattern ensures that even during massive repository re-scans, data corruption is avoided and the API remains responsive.

#### 3\. Semantic Intelligence: LightRAG and Knowledge Graph Integration

The transition from raw metadata to semantic intelligence is facilitated by  **LightRAG (version 1.4.9.11)** . Unlike standard vector search, which treats code as a collection of isolated text chunks, a Knowledge Graph approach preserves the multi-dimensional web of relationships within a codebase. This allows AI agents to understand not just what a file contains, but how it functions within the broader architecture.

##### Specialist Roles and Integration

The population of the Knowledge Graph is driven by specialized AI roles:

* **Librarian:**  Responsible for generating code summaries, indexing documentation, and detecting recurring patterns.  
* **Analyzer:**  Performs the deep structural analysis and summarization of components extracted by tree-sitter.The integration stores entity summaries and document chunks, connected by  **Graph Edges**  that track semantic relationships including  *Imports, Exports, Inherits, Calls, Implements,*  and  *Uses* .

##### Graceful Degradation and Offline Functionality

Docrunch is designed to maintain value even when specialized infrastructure is unavailable:

* **Query Engine Fallback:**  If LightRAG is offline, the system automatically falls back to  **Postgres full-text search** , utilizing tsvector columns to provide ranked results and snippets.  
* **Scanner Offline Mode:**  By utilizing the \--offline flag, the scanner bypasses all external databases and LLMs entirely, persisting metadata to a local .docrunch/offline\_files.json and generating basic Markdown output.This dual-path approach ensures that impact analysis—such as determining the "blast radius" of a change—remains possible regardless of environment constraints.

#### 4\. The Outbox Pattern: Ensuring Data Consistency and Synchronization

A primary technical challenge in hybrid architectures is maintaining consistency across Postgres, LightRAG, and Markdown files. Docrunch solves this using the  **Outbox Pattern** , ensuring that all storage backends remain eventually consistent without the performance penalties of distributed transactions.

##### Sync Behavior and the Persistence Flow

The synchronization logic is anchored in the sync\_events and storage\_sync tables. When a scan occurs or a report is filed, Postgres records the data and a corresponding sync event within a single atomic transaction. This guarantees that if the primary write fails, no orphaned sync events are created.

##### Failure Handling Logic

The system prioritizes backends based on their role as either source or derivative:

* **Postgres Failure:**  Immediate halt. The system cannot function without its relational anchor.  
* **LightRAG or Markdown Failure:**  The operation proceeds, and the failure is logged in storage\_sync. A dedicated Sync Worker then employs an exponential backoff retry policy to reconcile the state once the service is restored.

##### Idempotency via Content Hashing

To prevent redundant indexing and excessive disk I/O, Docrunch utilizes a  **Content Hash Idempotency**  mechanism. By comparing hashes of file content and documentation chunks, the system skips synchronization for data that has remained unchanged since the last successful sync, significantly optimizing performance for incremental updates.

#### 5\. Job Orchestration: Dual-Layer Execution with Redis and Temporal

Repository management involves a wide spectrum of latencies. Docrunch addresses this via a two-tier orchestration layer that routes tasks based on duration and durability requirements.

##### Job Router Logic

The  **Job Router**  serves as the traffic controller for the system, applying a 30-second threshold to determine execution paths:

* **Redis / Dramatiq Tier:**  Optimized for "Fast Tasks" (under 30s) such as simple metadata updates or individual file parses.  
* **Temporal Tier:**  Reserved for durable, long-running workflows that require complex state management and high reliability.

##### Temporal Workflows and the Saga Pattern

Core system activities are encapsulated in specific Temporal workflow classes:

* RepoScanWorkflow: Orchestrates the full repository analysis pipeline.  
* AITaskWorkflow: Manages multi-step agentic operations and task transitions.  
* MultiStepAnalysisWorkflow: Coordinates complex, cross-module codebase reasoning.These workflows utilize the  **Saga Pattern**  for error recovery. For example, if a workflow fails while writing a Markdown file, the system can execute a "compensation" activity to  **roll back a partially written vector index entry**  or clean up orphaned Postgres records, ensuring no "half-done" states persist.

##### Critical Takeaways for Job Resumability

1. **Step-Level Persistence:**  Progress is recorded at the activity level; interrupted scans resume exactly where they left off.  
2. **Deterministic Execution:**  Workflows are designed for safe replays by pushing all side effects (I/O) into activities.  
3. **Unified Observability:**  Redis jobs and Temporal workflows emit events to a unified bus, visualized in real-time via the Dashboard’s  **Workflow Visualizer** .

#### 6\. API Boundaries and Transport Layer Orchestration

The Docrunch architecture is fronted by a unified core service layer, exposed through two distinct transport facades to serve both human users and AI agents.

##### Facade Separation

* **REST (HTTP):**  A thin facade serving the React-based Dashboard, providing endpoints for search, task management, and LLM configuration.  
* **MCP (stdio):**  A bidirectional transport designed for AI coding agents. It enables a tool-based interaction model where agents use "Query Tools" to read context and  **"Report Tools"**  to write results.

##### Bidirectional Tooling and Report Logic

The MCP interface is the primary driver of the Outbox Pattern via high-impact report tools. When an agent uses record\_decision, log\_finding, claim\_task, or submit\_completion, these tools write directly to the Postgres task and history tables. This triggers the sync pipeline to update derived documentation and vector stores.

##### CLI Bridge and Proxy Architecture

To support local development and containerized environments, the  **CLI Bridge**  allows Docrunch to utilize local LLM binaries (Gemini, Claude, Codex). In Docker-based deployments, a  **Proxy Mode**  creates a tunnel to the host machine, enabling the containerized app to reach host-side LLMs via a specialized HTTP relay.

##### Real-Time Observability

Transport boundaries are augmented by  **WebSocket event streams** , providing live observability into the orchestration engine. These streams transmit lineage events, sync status updates, and real-time workflow progress, providing a professional-grade monitoring experience for complex, distributed AI workflows.  
