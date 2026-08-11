# Axion-UI

Monorepo for the **Axion** industrial-IoT condition-monitoring platform and its full
DevOps pipeline (CI/CD, image scanning, Helm, Argo CD GitOps).

## Repositories inside

| Component | Folder | Stack |
|-----------|--------|-------|
| Dashboard | `axion-ui/` | React + Vite + TS, served by Nginx (SPA + `/api` proxy) |
| Telemetry query service | `axion-telemetry-query-service/` | FastAPI + asyncpg |
| Telemetry ingestion service | `axion-ingestion-service/` | FastAPI + asyncpg |
| Data simulator | `axion-data-simulator/` | Python worker, POSTs telemetry |
| Database schema | `axion-database-schema/` | PostgreSQL migrations (SQL) |
| GitOps / Argo CD | `gitops/` | App-of-apps, Applications, Postgres chart |

## Architecture

```
Refinery devices (simulated) ──► axion-ingestion-service ──► PostgreSQL 16
                                              ▲                │
                                     axion-data-simulator      │
                                              │                ▼
        Browser ──► axion-ui (Nginx) ──► axion-telemetry-query-service
```

## Run everything locally (Docker)

The database runs as a Docker image (`postgres:16-alpine`); the schema is applied
automatically on first boot from the `axion-database-schema` image.

```bash
cp .env.example .env        # optional, defaults are fine
docker compose up --build
```

| Service | URL |
|---------|-----|
| Dashboard | http://localhost:8080 |
| Query API / docs | http://localhost:8000/docs |
| Ingestion API / docs | http://localhost:8001/docs |
| PostgreSQL | localhost:5432 |

## CI/CD

GitHub Actions (`.github/workflows/`) runs for all five services via a matrix:

1. `code-quality` — **SonarQube** scan + quality gate per service (blocking; Checkmarx/Blackduck hooks available)
2. `build-scan-push` — reusable pipeline: multi-stage Docker build → **Trivy** image scan (fails on HIGH/CRITICAL, SARIF → GitHub Security) → push to **ghcr.io/santandevops01012025/<service>**

Images are pushed on `main`/tags; pull requests get build + scan only.

## Kubernetes / Argo CD

Everything is deployed to Kubernetes via **Argo CD** from `gitops/`:

- `gitops/argocd/app-of-apps.yaml` — parent application
- `gitops/argocd/apps/*.yaml` — one Application per service (Helm charts)
- `gitops/db/` — PostgreSQL StatefulSet seeded from `axion-database-schema`

Bootstrap steps: see `gitops/README.md`.

## Secrets (GitHub Actions)

Repository secrets: `SONAR_HOST_URL`, `SONAR_TOKEN`. Optional commercial scanners:
`CHECKMARX_*`, `BLACKDUCK_*`, `PRISMA_*` (commented in the workflows).