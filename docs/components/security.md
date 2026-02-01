# Security Model

## Overview

Security considerations for Docrunch components.

> [!NOTE]
> Security features are implemented incrementally. Phase markers indicate when each feature becomes available.

---

## MCP Server Security

### Agent Identity

Each connecting agent should identify itself:

```python
class AgentIdentity:
    agent_id: str      # Unique ID (e.g., "claude-cli-1")
    agent_type: str    # claude, gemini, codex
    session_id: str    # Current session
```

### Authentication

**MVP / Phase 1 (Local):** No authentication required for localhost. If MCP binds beyond localhost, enable auth.

**Phase 3+ (Network):** Token-based authentication:

```yaml
server:
  mcp:
    auth:
      enabled: true
      tokens:
        - name: claude-agent
          token_env: DOCRUNCH_AGENT_TOKEN
          permissions: [read, write]
```

### Deployment Modes

**Local / Docker (MVP):**

- Bind MCP to localhost by default
- Auth disabled, write tools allowed for local agents
- Write tools can be disabled via config for read-only mode

**Hosted (VPS/Cloud):**

- Require token auth and role permissions for write tools
- Bind to network interfaces only behind TLS proxy or VPN
- Store tokens in environment variables or secret manager

### Rate Limiting

Prevent runaway agents:

```yaml
server:
  mcp:
    rate_limit:
      queries_per_minute: 60
      writes_per_minute: 30
      max_tokens_per_request: 10000
```

### Permissions

Role-based access:

| Role   | Read | Write | Admin |
| ------ | ---- | ----- | ----- |
| reader | yes  | no    | no    |
| agent  | yes  | yes   | no    |
| admin  | yes  | yes   | yes   |

---

## Dashboard Security (Phase 4+)

### Authentication

JWT-based authentication:

```python
class DashboardAuth:
    def login(username: str, password: str) -> JWTToken
    def verify_token(token: str) -> User
    def refresh_token(token: str) -> JWTToken
```

REST auth endpoints (login/refresh/logout) are planned for Phase 4+ and are not
part of the MVP. Local dev assumes localhost-only access or an external auth
proxy.

### Session Management

- Token expiry: 24 hours
- Refresh token: 7 days
- Secure cookie storage

---

## Secrets Handling

### Environment Variables

All secrets via environment:

```python
# Correct
api_key = os.environ.get("OPENAI_API_KEY")

# Wrong
api_key = "sk-..."  # Never hardcode
```

### Configuration

Config references env vars, not values:

```yaml
llm:
  providers:
    openai:
      api_key_env: OPENAI_API_KEY # Reference, not value
```

### LLM Provider Settings

- Provider credentials are stored in Postgres, encrypted at rest.
- UI displays masked secrets and only allows updates for admin roles.
- Encryption key is supplied via environment (e.g., `DOCRUNCH_KMS_KEY`).
- Default bootstrap can load env vars into Postgres for local dev.
- CLI Bridge proxy keys can be set per provider config or via `CLI_PROXY_API_KEY`.

### .env Support

For local development:

```bash
# .env (git-ignored)
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-...
CLI_PROXY_API_KEY=your-proxy-key
```

---

## Audit Logging (Phase 5+)

### What to Log

| Event          | Data                                 |
| -------------- | ------------------------------------ |
| Agent connect  | agent_id, timestamp                  |
| Query executed | agent_id, query, timestamp           |
| Task claimed   | agent_id, task_id, timestamp         |
| Task completed | agent_id, task_id, result, timestamp |
| Error occurred | agent_id, error, timestamp           |

### Log Format

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "event": "query_executed",
  "agent_id": "claude-cli-1",
  "data": {
    "query": "search_docs",
    "params": { "query": "auth patterns" },
    "duration_ms": 150
  }
}
```

---

## Data Protection

### Sensitive Code

Docrunch indexes code which may contain secrets. Mitigations:

1. **Pattern detection** - Warn on potential secrets in code
2. **Exclusions** - `.docrunch/ignore.yaml` for sensitive paths
3. **Raw source handling** - Process source in memory and persist only metadata and derived summaries by default; optional snippet storage must be explicitly enabled.

### Vector Storage

LightRAG stores embeddings and derived summaries, not raw source by default.

- Postgres stores metadata, hashes, and derived summaries
- Raw source storage is off by default (opt-in for limited snippets)
- Embeddings can be purged independently

---

## Network Security

### Local Mode (Default)

```yaml
server:
  mcp:
    host: localhost # Only local connections
    port: 3333
```

### Network Mode

If exposing to network:

```yaml
server:
  mcp:
    host: 0.0.0.0
    port: 3333
    tls:
      enabled: true
      cert: /path/to/cert.pem
      key: /path/to/key.pem
```

---

## Security Checklist

- [ ] No secrets in code
- [ ] All API keys from environment
- [ ] Rate limiting enabled
- [ ] Audit logging enabled
- [ ] localhost-only by default
- [ ] TLS for network exposure
