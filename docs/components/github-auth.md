# GitHub Authentication

Configure GitHub access for cloning, API features, and remote repository management.

## Authentication Options

### Option 1: Local Git Only (Default)

**Best for:** Personal development, local-only workflows

No GitHub auth needed. Uses your existing Git credentials (SSH keys or credential manager).

| Feature                         | Supported |
| ------------------------------- | --------- |
| Push/pull via Git               | ✅        |
| Clone with existing credentials | ✅        |
| GitHub API (issues, PRs)        | ❌        |
| Remote clone via Docrunch       | ❌        |

---

### Option 2: Personal Access Token (Recommended)

**Best for:** Personal use with full GitHub API access

**Setup:**

1. Create token at: https://github.com/settings/tokens
2. Select scopes: `repo`, `read:user`
3. Configure in Docrunch:

```yaml
# .docrunch/config.yaml
github:
  auth_type: pat
  token: ${GITHUB_TOKEN} # From environment variable
```

Or store encrypted via CLI:

```bash
docrunch config set github.token ghp_xxxx
```

**Features enabled:**

- Clone private repos
- View issues and PRs in dashboard
- List all user repositories
- API rate: 5000 requests/hour

---

### Option 3: OAuth App

**Best for:** Multi-user or web-hosted deployments

**Setup:**

1. Create OAuth App at: https://github.com/settings/developers
2. Set callback URL: `http://localhost:8080/auth/github/callback`
3. Configure:

```yaml
# .docrunch/config.yaml
github:
  auth_type: oauth
  client_id: ${GITHUB_CLIENT_ID}
  client_secret: ${GITHUB_CLIENT_SECRET}
```

**Features enabled:**

- User-specific permissions
- "Sign in with GitHub" flow
- Organization repository access
- Multi-user support

---

### Option 4: GitHub App

**Best for:** Organization-wide deployment, webhooks

Most complex setup. Enables:

- Fine-grained permissions per repo
- Webhooks for event-driven updates (auto-scan on push)
- Higher API rate limits (5000/hour per installation)
- Organization-wide installation

See GitHub's [Creating a GitHub App](https://docs.github.com/en/apps/creating-github-apps) guide.

---

## Recommendation

| Use Case                      | Recommended Auth |
| ----------------------------- | ---------------- |
| Personal dev, single machine  | PAT              |
| Team with shared server       | OAuth App        |
| Organization-wide             | GitHub App       |
| No GitHub needed (local only) | None (default)   |

## Database Schema

```sql
CREATE TABLE github_auth (
    id UUID PRIMARY KEY,
    auth_type VARCHAR(20) NOT NULL,  -- pat, oauth, app
    token_encrypted TEXT,
    client_id TEXT,
    client_secret_encrypted TEXT,
    installation_id TEXT,            -- For GitHub App
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

## API Endpoints

| Endpoint                 | Description                  |
| ------------------------ | ---------------------------- |
| `GET /api/github/repos`  | List accessible repositories |
| `GET /api/github/issues` | List issues for active repo  |
| `GET /api/github/prs`    | List PRs for active repo     |
| `POST /api/github/clone` | Clone a remote repository    |
