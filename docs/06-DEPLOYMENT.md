# Corebridge — Deployment & CI/CD

---

## Infrastructure per Bank

```
Bank's On-Prem Server(s)
│
├── Docker Host
│   ├── corebridge-api          ← NestJS backend (2+ instances)
│   ├── corebridge-backoffice   ← React app served by Nginx
│   ├── postgresql              ← Database
│   ├── minio                   ← Object storage
│   ├── grafana                 ← Monitoring dashboards
│   └── prometheus              ← Metrics collection
│
├── ICS BANKS Core Server       ← Bank's existing core (SOAP)
│
└── Network
    ├── Load balancer (Nginx or HAProxy)
    ├── Internal network only (no public internet for core)
    └── Mobile app → LB → corebridge-api → Core
```

---

## Docker Compose

```yaml
version: '3.8'

services:
  api:
    build: ./apps/api
    ports: ["3000:3000"]
    environment:
      - DATABASE_URL=postgresql://corebridge:secret@postgres:5432/corebridge
      - ICSFS_WSDL_URL=http://core-server:8080/wsdl
      - KYC_PROVIDER=sumsub
      - STORAGE_BACKEND=minio
      - JWT_SECRET=${JWT_SECRET}
      - NODE_ENV=production
    depends_on: [postgres, minio]
    restart: always
    deploy:
      replicas: 2

  backoffice:
    build: ./corebridge-backoffice
    ports: ["8080:80"]
    restart: always

  postgres:
    image: postgres:16
    volumes: [pg_data:/var/lib/postgresql/data]
    environment:
      - POSTGRES_DB=corebridge
      - POSTGRES_USER=corebridge
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    restart: always

  minio:
    image: minio/minio
    volumes: [minio_data:/data]
    environment:
      - MINIO_ROOT_USER=${MINIO_USER}
      - MINIO_ROOT_PASSWORD=${MINIO_PASSWORD}
    command: server /data
    restart: always

  grafana:
    image: grafana/grafana
    ports: ["3001:3000"]
    volumes:
      - grafana_data:/var/lib/grafana
      - ./infra/grafana/dashboards:/etc/grafana/provisioning/dashboards
    restart: always

  prometheus:
    image: prom/prometheus
    volumes: [./infra/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml]
    restart: always

volumes:
  pg_data:
  minio_data:
  grafana_data:
```

---

## CI/CD Pipeline

```
Developer pushes to corebridge-backend
    │
    ▼
GitHub Actions
    ├── 1. Lint + Type Check
    ├── 2. Run integration tests
    ├── 3. Validate OpenAPI spec matches Swagger output
    ├── 4. Build all @corebridge/* packages
    ├── 5. Publish to GitHub Packages
    ├── 6. Build Docker image
    └── 7. Push to private Docker registry

Bank deployment:
    ├── Bank's CI pulls latest @corebridge/* packages
    ├── Bank's repo builds with customizations
    ├── Docker image built with bank-specific config
    └── Deployed to bank's on-prem via SSH / Ansible / manual
```

---

## Environments

| Environment | Purpose | Where |
|-------------|---------|-------|
| Local dev | Development | Developer's machine (docker-compose) |
| Staging | Pre-deployment testing | Corebridge office or cloud VM |
| Bank staging | Integration testing | Bank's on-prem (test core) |
| Bank production | Live | Bank's on-prem (live core) |

---

## Secrets

```
JWT_SECRET           ← Access token signing
JWT_REFRESH_SECRET   ← Refresh token signing
DB_PASSWORD          ← PostgreSQL
MINIO_USER / MINIO_PASSWORD
ICSFS_USERNAME / ICSFS_PASSWORD
KYC_API_KEY          ← Sumsub/Regula
SANCTIONS_API_KEY
FCM_SERVER_KEY       ← Firebase push
ENCRYPTION_KEY       ← AES-256 for PII
```

Managed via Docker secrets or `.env` files mounted as volumes.

---

## Backups

| What | How | Frequency |
|------|-----|-----------|
| PostgreSQL | pg_dump to encrypted file | Daily |
| MinIO objects | Sync to secondary storage | Daily |
| Docker volumes | Volume backup scripts | Daily |
| Configuration | Git (version controlled) | Every change |

Corebridge provides scripts and cron jobs. Bank executes them.

---

## Health Checks

```
GET /health              ← Basic: API is running
GET /health/detailed     ← DB, core, storage, KYC provider status
```

Grafana + Prometheus monitors these.
