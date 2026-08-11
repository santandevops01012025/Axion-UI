# Axion Telemetry Query Service

FastAPI service powering the Axion condition-monitoring dashboard. Reads telemetry from PostgreSQL (asyncpg) and exposes dashboard/device endpoints.

## API

| Endpoint | Description |
|----------|-------------|
| `GET /health` | Liveness/readiness probe |
| `GET /dashboard/summary` | Online assets + last update |
| `GET /dashboard/throughput` | Throughput per minute (last 60m) |
| `GET /dashboard/regions` | Region summary (devices/online/alerts) |
| `GET /devices` | Latest state per device with health status |
| `GET /devices/{id}/latest` | Latest telemetry for a device |
| `GET /devices/{id}/trends?hours=N` | Historical trends |
| `GET /devices/top-anomalous` | Top anomalous devices |

## Local run

```bash
pip install -r requirements.txt
export DATABASE_URL=postgresql://axion_user:P%40ssw01rd%40123@localhost:5432/axion_db
uvicorn main:app --reload --port 8000
```

## 🔄 CI/CD & Deployment

- **Multi-stage Dockerfile** (wheels only in runtime stage, non-root user, `/health` probe)
- **Pipeline** (`.github/workflows/ci.yml`): SonarQube quality gate → build → **Trivy** scan (fails on HIGH/CRITICAL) → push to **ghcr.io/devopsinsiders/axion-telemetry-query-service**
- **Helm chart**: `helm/axion-telemetry-query-service` (Deployment + Service + HPA, DB DSN from secret `axion-db-credentials`)
- **Argo CD**: managed by the `axion-telemetry-query-service` Application in [axion-gitops](https://github.com/devopsinsiders/axion-gitops)