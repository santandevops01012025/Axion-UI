# Axion GitOps

Argo CD control-plane for the Axion platform. Lives inside the Axion-UI monorepo.

```
axion-ui/helm/axion-ui                                    ─► dashboard (Nginx/React)
axion-telemetry-query-service/helm/...                    ─► query API
axion-ingestion-service/helm/...                          ─► ingestion API
axion-data-simulator/helm/...                             ─► simulator worker
axion-database-schema/helm/...                            ─► schema migration Job
gitops/db/                                                ─► PostgreSQL StatefulSet
gitops/argocd/app-of-apps.yaml                            ─► parent Application
gitops/argocd/apps/*.yaml                                 ─► one Application each
```

Every service's `.github/workflows/ci.yml` gates (SonarQube), scans (Trivy,
fail on HIGH/CRITICAL) and pushes its image to **ghcr.io**; Argo CD syncs the
Helm charts from this repo.

## Bootstrap

```bash
# 1. Register this repo in Argo CD (use a PAT if it is private)
argocd repo add https://github.com/santandevops01012025/Axion-UI.git \
  --username <user> --password <PAT> --upsert

# 2. Create the app project
kubectl apply -n argocd -f gitops/argocd/project.yaml

# 3. Deploy the app-of-apps (creates all child Applications)
kubectl apply -n argocd -f gitops/argocd/app-of-apps.yaml

# 4. Watch
argocd app list
kubectl get pods -n axion -w
```

## Database

- `gitops/db/` deploys `postgres:16-alpine` (StatefulSet + PVC)
- On first boot an initContainer copies the validated `*.sql` from the
  `axion-database-schema` image into `/docker-entrypoint-initdb.d/`
- Secret `axion-db-secret` holds the credentials; `axion-db-credentials`
  exposes the derived `DATABASE_URL` DSN to every service

## Common tasks

```bash
argocd app set axion-telemetry-query-service --helm-set image.tag=sha-<sha>
argocd app sync axion-ui
argocd app rollback axion-ingestion-service <revision>
```

## Private GHCR packages

If images are private the cluster needs a pull secret on every workload:

```bash
kubectl -n axion create secret docker-registry ghcr-pull \
  --docker-server=ghcr.io --docker-username=<GH_USER> --docker-password=<PAT with read:packages>
argocd app set axion-ui --helm-set imagePullSecrets[0].name=ghcr-pull
```

## Promotion rules

Prod promotion requires: **SonarQube gate green** and **Trivy clean
(no HIGH/CRITICAL)**. Findings that are base-image-only or have no upstream fix
are triaged by DevOps/AppSec and recorded in each repo's `.trivyignore` with an
expiry date — the rest (~70%) is fixed by developers.