# Docker (Local Infra)

Docrunch runs on the host during development. A docker-compose stack runs Postgres, Redis, and LightRAG in containers (no local LightRAG install) without containerizing the Docrunch app.

## What's prepared

- `docker-compose.yml` with:
  - Postgres 16 on port 5430 (db/user/password: docrunch)
  - Redis 7 on port 6370
- LightRAG on port 9621 (`ghcr.io/hkuds/lightrag:latest`)
- Named volumes for persistence.

## Start and stop

```bash
docker compose up -d
docker compose ps
docker compose down
```

## App configuration

- `storage.postgres_url`: `postgresql://docrunch:docrunch@localhost:5430/docrunch`
- `jobs.redis_url`: `redis://localhost:6370/0`
- LightRAG base URL: `http://localhost:9621` (used by the LightRAG integration when implemented)

## Notes

- Use `docker compose down -v` to wipe local data.
- The Docrunch app is not containerized yet; only infra services are.
