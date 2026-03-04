# gallery-api

A production-grade Go REST API built to exercise the SRE contract:
`/healthz`, `/readyz`, `/metrics`, and full distributed tracing.

This service is the primary workload in the SRE lab platform. Its health, latency,
and error rate drive the SLO dashboards, burn-rate alerts, and incident drills.

## What This Service Does

A photo gallery REST API backed by PostgreSQL. The business logic is intentionally
simple so that 100% of engineering focus goes to the **operational contract** — the
things an SRE cares about.

## The SRE Contract

| Endpoint | Purpose | Monitored By |
|---|---|---|
| `GET /healthz` | Liveness probe — is the process alive? | kubelet → restart if failing |
| `GET /readyz` | Readiness probe — is the DB connected? | kubelet → remove from Service endpoints |
| `GET /metrics` | Prometheus scrape endpoint | Prometheus → Grafana RED dashboard |

## Observability

**OpenTelemetry** instrumented on every HTTP handler. Traces flow to:
- **Tempo** (in-cluster) for distributed trace storage
- **DataDog APM** via DD agent DaemonSet on port 8126

**Prometheus metrics** exported from `/metrics`:

| Metric | Type | Purpose |
|---|---|---|
| `http_requests_total` | Counter | Request rate, labeled by method + status code |
| `http_request_duration_seconds` | Histogram | Drives SLO burn-rate alerts |
| `db_connections_active` | Gauge | PostgreSQL connection pool health |

## SLO Definition

| SLO | Target | Consequence |
|---|---|---|
| Availability (non-5xx / total) | 99.9% over 30 days | 14× burn rate → PagerDuty call |
| Latency p99 < 500ms | 99% of requests | k6 pipeline gate — blocks promotion |

## Deployment

Deployed via GitOps to 4 isolated Kubernetes namespaces managed by
[sre-lab-infra](https://github.com/clementokoro/sre-lab-infra):

| Environment | Replicas | Purpose |
|---|---|---|
| `dev` | 1 | Development, debug logging enabled |
| `qa` | 1 | Integration test target, test database |
| `staging` | 2 | Mirrors production config, k6 load test target |
| `production` | 2 | Strict NetworkPolicy, full alerting active |

Each environment is an ArgoCD Application managed by a Hub & Spoke ApplicationSet.
Image promotion from staging → production requires all pipeline gates to pass.

## Container Build

Multi-stage Docker build producing a **distroless final image** — no shell, no
package manager, minimal attack surface.
```dockerfile
# Stage 1 — build
FROM golang:1.24 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o gallery-api ./cmd/api

# Stage 2 — minimal runtime
FROM gcr.io/distroless/static:nonroot
COPY --from=builder /app/gallery-api /gallery-api
ENTRYPOINT ["/gallery-api"]
```

## CI/CD Pipeline

5-stage GitLab CI/CD pipeline on every merge request:
```
build → test → security-scan → deploy-staging → load-test
```

| Stage | Gate |
|---|---|
| **security-scan** | Trivy — pipeline fails on any CRITICAL CVE |
| **load-test** | k6 100 VUs — fails if p99 latency > 500ms |

SLSA Level 2 provenance generated on every build via GitLab CI artifact attestation.

## Local Development
```bash
# Build
go build -o gallery-api ./cmd/api

# Run (requires PostgreSQL)
export DB_HOST=localhost DB_PORT=5432 DB_NAME=gallery DB_USER=gallery DB_PASS=secret
./gallery-api

# Verify SRE contract endpoints
curl http://localhost:8080/healthz   # → {"status":"ok"}
curl http://localhost:8080/readyz    # → {"status":"ok","db":"connected"}
curl http://localhost:8080/metrics   # → Prometheus exposition format
```
