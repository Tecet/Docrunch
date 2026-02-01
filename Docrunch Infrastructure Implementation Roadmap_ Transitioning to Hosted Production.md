### Docrunch Infrastructure Implementation Roadmap: Transitioning to Hosted Production

#### 1\. Strategic Infrastructure Overview

The transition of Docrunch from a local Docker-based development environment to a professional hosted infrastructure is an architectural necessity. While local stdio transports facilitate rapid prototyping, they represent a significant single-point-of-failure and lack the throughput required for complex multi-agent orchestration. To scale Docrunch into its role as a persistent "Project Manager" for AI agents, we must move toward a hosted Virtual Private Server (VPS) or cloud-native model. This shift ensures 24/7 availability, multi-user collaboration, and the asynchronous consistency required for high-volume repository analysis.The following table summarizes the infrastructure delta as we move from development to production:| Service | Local Development (Docker Compose) | Hosted Production (VPS/Cloud) | Persistence & Connectivity || \------ | \------ | \------ | \------ || **Postgres** | Port 5430 (localhost) | Port 5430 (Internal Network) | Managed instances; volume-backed persistence. || **Redis** | Port 6370 (localhost) | Port 6370 (Internal Network) | Ephemeral job state; managed cache services. || **LightRAG** | Port 9621 (localhost) | Port 9621 (Internal API) | Vector/Knowledge Graph persistence via named volumes. || **Temporal** | Port 7233 (localhost) | Port 7233 (Service), Port 8080 (UI) | Orchestration namespace docrunch; high-availability. || **Transports** | Local stdio / HTTP | TLS-Terminated REST / MCP-over-HTTP | Secure network ingress via JWT-authenticated proxies. |  
This foundation enables a persistent state for AI specialists, allowing the platform to manage long-running tasks that extend beyond the lifecycle of a single developer session. The cornerstone of this transition is the hardening of our credential orchestration.

#### 2\. Secure Credential Orchestration and Identity Management

As the platform egresses from the local machine, the strategic importance of protecting LLM API keys and database credentials cannot be overstated. We must transition from simple environment-based loading to a hardened orchestration model that prioritizes secret rotation and at-rest encryption.

##### Credential Encryption and Pydantic Validation

Docrunch utilizes the DOCRUNCH\_KMS\_KEY for at-rest encryption of sensitive blobs within the llm\_providers Postgres table. A critical differentiator in our architecture is the use of a Pydantic-validated settings system. Unlike traditional bootstrap methods, this system performs strict type-safety and validation checks upon loading. This prevents "cold-start" failures where invalid or malformed credentials cause runtime crashes during expensive LLM provider calls. All secrets are decrypted only at the service layer immediately before egress to the provider.

##### AI Agent Identity and Auditability

In a hosted environment, local stdio trust is replaced by token-based identity. Every AI agent interacting with the MCP server must possess a unique identifier to ensure absolute auditability across the task lifecycle. To maintain architectural integrity and prevent state drift, the following fields are mandatory for agent reporting:

* **claim\_task** : task\_id, reported\_by (Unique Agent ID), reported\_at (ISO Timestamp).  
* **update\_progress** : task\_id, percentage (integer 0-100), status, reported\_by, reported\_at.  
* **submit\_completion** : task\_id, summary, changes, tests\_passed (Boolean), reported\_by, reported\_at.Securing identity at the application layer is the prerequisite for establishing safe network-level transport boundaries.

#### 3\. Network-Level Security and Transport Boundaries

Exposing the Model Context Protocol (MCP) and REST transports to a public or private network introduces significant ingress risks. We mitigate these through strict interface binding and the enforcement of transport boundaries.

##### Comparative Transport Configurations

Hosted Docrunch requires a departure from stdio in favor of network-ready interfaces:

* **MCP (stdio)** : Remains the default for local CLI use, but is disabled in cloud environments.  
* **REST/HTTP** : The primary interface for the React Dashboard. In "Hosted Mode," the MCP transport must be reconfigured to server.mcp.transport: http. This setup requires a TLS-terminating proxy (e.g., Nginx) to secure data in transit and manage SSL certificates.

##### Hosted Security and Rate Limiting

To protect against "runaway agents" that could cause catastrophic API cost spikes, the hosted roadmap enforces the following:

* **Rate Limiting** : Configured under the server.mcp block, limits are enforced per agent to monitor and throttle excessive requests.  
* **Role-Based Tool Visibility** : In production, the "reader" role must be restricted to zero write access. Tools such as record\_decision or submit\_completion are only visible and executable by authorized roles.  
* **JWT Authentication** : The React Dashboard must utilize JSON Web Tokens (JWT) for all REST interactions, ensuring that only authenticated sessions can access the internal repository state.

#### 4\. Production-Grade Storage and Job Orchestration

The reliability of a hosted Docrunch instance depends on its "Source of Truth" (Postgres) and the asynchronous consistency maintained by our Outbox Pattern and job orchestration layers.

##### The Outbox Pattern and Storage Sync

Postgres serves as the authoritative store for all structured metadata. To ensure that derived stores like LightRAG (vector graphs) and Markdown documentation never drift from reality, Docrunch implements the  **Outbox Pattern** . Any change in Postgres emits an event to the sync\_events table within the same transaction. The SyncWorker then processes these events, ensuring idempotency through content hashes. If a derived store is unavailable, the system remains operational, and consistency is restored via the docrunch sync rebuild command.

##### Job Orchestration: Redis vs. Temporal

We utilize a two-layer job system to optimize for both speed and durability:

* **Fast Tasks (Redis/Dramatiq)** : Tasks with an estimated duration \< 30 seconds are routed to Redis for immediate execution.  
* **Durable Workflows (Temporal)** : Long-running operations, such as repository scans (RepoScanWorkflow) or multi-step analysis, are routed to the Temporal service (Port 7233).Architects must ensure the docrunch namespace is initialized via Temporal admin tools. Temporal’s implementation of the  **Saga Pattern**  is critical for production; it allows for automatic compensation (rollback) logic if a complex analysis fails, maintaining the integrity of the knowledge base. The Temporal Web UI (Port 8080\) must be secured and monitored to track workflow health.

#### 5\. Remote CLI Bridge and Proxy Deployment

For Dockerized or cloud-hosted instances, direct access to host-resident CLI tools (Gemini, Claude, Codex) is blocked by the container boundary. The  **CLI Bridge Proxy**  acts as a secure tunnel to resolve this.

##### Technical Guide for Proxy Deployment

The scripts/cli\_proxy.py must be deployed on the host machine where the CLI tools are installed.

1. **Authentication** : Secure the connection by configuring the CLI\_PROXY\_URL and X-API-Key in the Docrunch LLM provider settings.  
2. **Discovery Logic** : The proxy performs executable discovery by searching the system PATH via shutil.which, with fallbacks to common install locations like AppData\\Roaming\\npm (Windows) or /usr/local/bin (Linux/macOS).  
3. **Model Registry** : Remote models must be registered in .docrunch/cli\_models.json.**Example Registry Structure:**

{  
  "gemini": \[  
    {  
      "id": "gemini-1.5-pro",  
      "display\_name": "Gemini 1.5 Pro",  
      "description": "High-reasoning model for complex repository analysis"  
    }  
  \]  
}

The CLI Bridge's health checks (GET /health) are integrated into the primary Docrunch observability stream, ensuring tool availability is transparent to the orchestrator.

#### 6\. Implementation Phasing and Observability

The transition to production must be handled in phases, supported by deep observability to prevent and debug state drift between the relational and semantic stores.

##### Production Readiness Checklist

*   **Validate Alembic Migration State** : Ensure the Postgres schema is current before any data operations.  
*   **Audit Logging Enabled** : Confirm all agent connections and submit\_completion events are logged.  
*   **Cost Tracking & Budget Alerts** : Configure thresholds to prevent runaway LLM usage.  
*   **Snapshot creation** : Execute a full system snapshot using the Rollback System before major migrations.  
*   **Network Egress Lockdown** : Verify that all services are bound to internal interfaces and TLS proxies are active.

##### Data Flow Visualization and State Drift

Observability in Docrunch is not merely for monitoring; it is a debugging tool for asynchronous consistency. The React Flow-based visualizers serve as the primary interface:

* **Lineage Graph** : Directly visualizes the Outbox Pattern, showing the flow from Postgres to SyncWorker to LightRAG.  
* **Storage Health** : Provides real-time status of Postgres (5430), Redis (6370), LightRAG (9621), and Temporal (7233/8080).  
* **Sync Event Visualizer** : Animates the sync\_events stream, allowing architects to identify and resolve processing bottlenecks.By adhering to this roadmap, we transform the Docrunch codebase from a local development tool into a hardened Enterprise Intelligence Platform, ready for multi-agent collaboration at scale.

