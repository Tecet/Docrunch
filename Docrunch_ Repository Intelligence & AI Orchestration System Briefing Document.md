### Docrunch: Repository Intelligence & AI Orchestration System Briefing Document

#### Executive Summary

Docrunch is a sophisticated Repository Intelligence and AI Orchestration System designed to transform static codebases into living, AI-accessible knowledge bases. The system primary addresses the "context loss" problem inherent in modern AI-assisted development, where coding assistants (such as Claude, Gemini, or Codex) lose track of project architecture and conventions between sessions.As of February 2026, the project is approximately  **60% complete** , having successfully established its core engine, storage infrastructure, and bidirectional Model Context Protocol (MCP) server. Docrunch utilizes a 2-layer LLM architecture, combining specialized "specialists" for analysis with a task-based orchestration engine for external coding agents. Key technical foundations include  **tree-sitter**  for AST parsing,  **Postgres**  for relational metadata, and  **LightRAG**  for graph-based semantic search. The system is currently focused on Phase 4, which introduces advanced React Flow-based visualizations for codebase structure and data lineage.

#### 1\. System Philosophy and Core Objectives

The fundamental premise of Docrunch is that repositories should serve as more than just a collection of source files; they must function as an intelligent partner in the development lifecycle.

##### The Problem: Context Decay

* **Inconsistency:**  AI assistants often forget patterns established in previous sessions, leading to fragmented and non-idiomatic code.  
* **Repetitive Onboarding:**  Developers spend excessive time re-explaining architecture to AI agents.  
* **Desynchronization:**  Documentation often lags behind the actual state of the code produced by AI.

##### The Solution: Persistent Intelligence

Docrunch maintains a persistent "memory" of the repository through:

1. **Deep Scanning:**  Analyzing not just text, but Abstract Syntax Trees (AST) to understand relationships.  
2. **Semantic Mapping:**  Preserving the semantic web of code through a knowledge graph.  
3. **Standardized Protocol:**  Utilizing MCP to allow agents to both query context and report findings back to the system.

#### 2\. Technical Architecture

Docrunch employs a multi-tiered architecture that ensures data integrity and operational robustness.

##### 2.1 Core Engine

The Core Engine manages the lifecycle of codebase understanding:

* **Scanner:**  Performs async directory walking, respecting .gitignore and .docrunch ignore patterns, while calculating file hashes for change detection.  
* **Parser:**  Uses  **tree-sitter**  to extract classes, functions, and imports with precise line spans for Python, TypeScript, and JavaScript.  
* **Analyzer:**  Detects complex relationships (inherits, calls, implements) and generates AI-powered summaries of components.  
* **Offline Mode:**  A critical fallback that skips Postgres/LLM dependencies to generate basic Markdown documentation using local JSON caches.

##### 2.2 Storage Layer and Synchronization

Docrunch utilizes a "Source of Truth" model with derived stores:

* **Postgres (Source of Truth):**  Stores structured metadata, tasks, LLM settings, and chat history.  
* **LightRAG (Derived):**  Provides vector storage and a knowledge graph for semantic search.  
* **Markdown (Derived):**  Generates human-readable documentation directly within the repository.  
* **Outbox Sync System:**  Ensures consistency by emitting events from Postgres that are processed by workers to update LightRAG and Markdown, preventing data drift.

##### 2.3 Background Job System

Operations are routed via a  **Job Router**  to two distinct backends:

* **Redis/Dramatiq:**  Handles "fast" tasks (e.g., metadata updates) with low latency (\<30s).  
* **Temporal:**  Manages "durable" workflows for long-running operations like full repository scans or batch embeddings, utilizing the  **Saga Pattern**  for automatic rollbacks on failure.

#### 3\. LLM Intelligence and Specialist Roles

Docrunch implements a 2-layer LLM architecture to separate repository management from specific task execution.

##### 3.1 Specialist Profiles (Layer 1\)

Specialists are internal roles managed via the Docrunch Dashboard: | Role | Purpose | | :--- | :--- | |  **Librarian**  | Manages documentation, RAG indexing, and pattern detection. | |  **Security**  | Focuses on threat modeling and secrets detection. | |  **QA**  | Handles test generation and coverage analysis. | |  **Analyzer**  | Responsible for code summarization and structural analysis. | |  **Chat**  | Powers the user-facing Q\&A interface. |

##### 3.2 CLI Bridge

The  **CLI Bridge**  allows Docrunch to utilize locally installed LLM CLI tools (Gemini, Claude, Codex) as providers.

* **Direct Mode:**  Executes CLIs via subprocess for local development.  
* **Proxy Mode:**  Uses a host-side proxy script to relay requests from dockerized environments to the host machine's CLI tools.

#### 4\. Agent Orchestration and MCP

The  **Model Context Protocol (MCP)**  is the primary transport for AI agents to interact with the repository.

##### 4.1 Bidirectional Communication

Unlike standard RAG systems, Docrunch’s MCP implementation is bidirectional:

* **Read (Query Tools):**  Agents use search\_docs, get\_component, and get\_relations to gain context.  
* **Write (Report Tools):**  Agents use claim\_task, log\_finding, record\_decision, and submit\_completion to update the repository state.

##### 4.2 Kanban Pipeline

AI work is managed through a structured pipeline:

* **Backlog:**  Tasks defined by type (Bug Fix, Feature, Refactor, Security).  
* **Working:**  Active agents claiming tasks and reporting progress.  
* **Review:**  Tasks awaiting human or Manager AI approval.  
* **Done:**  Completed tasks, with reports automatically synced to Markdown.

#### 5\. Visualizations and User Interface

The Dashboard, built with React and Vite, provides interactive tools for repository management.

##### 5.1 React Flow Visualizers

The system is currently deploying a suite of advanced visualizers:

* **Repository Graph:**  Displays codebase structure, dependencies, and component relationships.  
* **Workflow Visualizer:**  Real-time monitoring of Temporal and Redis job statuses.  
* **Knowledge Graph:**  Navigates semantic entities and LightRAG relationships.  
* **Data Lineage:**  (Queued) Visualizes the flow of data from scanner to final storage output.

##### 5.2 RAG-Powered Chat

Users can interact with the codebase via a chat interface that provides:

* **Context Scope:**  Ability to scope queries to documentation, agent reports, or both.  
* **Citations:**  Interactive "chips" that link answers directly to the source file and line number.

#### 6\. Implementation Status and Roadmap

##### 6.1 Current Milestone (M5 Completed)

The system has delivered the REST API, WebSocket streams, Dashboard scaffold, and basic knowledge/workflow graphs.

##### 6.2 Active and Queued Work

* **Active:**  TASK-016 (Task Manager) implementation.  
* **Phase 4 (Ongoing):**  Implementation of 33 queued visualization tasks, including impact analysis, complexity heatmaps, and collaborative cursors.  
* **Phase 5 (Backlog):**  Advanced features including  **Session Memory** ,  **Cost Tracking** ,  **Git Integration** , and  **Agent Analytics** .

##### 6.3 System Fallback Matrix

Component,Full Mode,Offline Mode  
Scanner/Parser,Active,Active  
Postgres/LightRAG,Active,Skipped  
LLM Summaries,AI-Generated,Template-Based  
Query,Semantic Search,Basic Text Search

#### 7\. Critical Insights and Authoritative Quotes

"Repositories aren't just code files anymore. They are living knowledge bases." —  *System Preview*"Docrunch fixes context loss by transforming your codebase into an intelligent partner... keeping documentation in sync with actual work." —  *Project Overview***Operational Mandate:**  "Postgres is the source of truth for all structured data. LightRAG and Markdown are derived stores... Failures are recorded in storage\_sync and retried by the sync worker." —  *Storage Sync Architecture***On AI Collaboration:**  "Party Mode allows multi-agent collaboration with explicit roster, shared context, and structured turn-taking... Dissent and open questions are captured as artifacts." —  *Advanced Features (Backlog)*  
