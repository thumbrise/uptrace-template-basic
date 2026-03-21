> **One `docker compose up` away from a full observability platform.**
> Traces, logs, metrics — collected, batched, stored, visualized.
> Multi-tenant ready. Zero hardcoded secrets. 16 env vars, fail-fast on every one.
---
## Architecture
```
┌─────────────────────────────────────────────────────────────────────┐
│  YOUR APPLICATIONS                                                  │
│  (any language / any OTel SDK)                                      │
│                                                                     │
│  DSN: http://<token>@localhost:14318?grpc=14317                     │
│  Header: uptrace-dsn                                                │
└──────────┬──────────────────────────────────┬───────────────────────┘
           │ gRPC :4317                       │ HTTP :4318
           ▼                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   OTel Collector  (contrib 0.123.0)                 │
│                                                                     │
│  ┌─────────────────────────┐   ┌──────────────────────────────┐    │
│  │  CLIENT PIPELINE        │   │  SYSTEM PIPELINE             │    │
│  │                         │   │                              │    │
│  │  receivers:             │   │  receivers:                  │    │
│  │    otlp (gRPC + HTTP)   │   │    hostmetrics (7 scrapers)  │    │
│  │                         │   │    postgresql/uptrace         │    │
│  │  processor:             │   │    redis/uptrace              │    │
│  │    batch/clients        │   │    httpcheck/self             │    │
│  │    10K batch / 10s      │   │    prometheus/self            │    │
│  │    metadata_keys:       │   │                              │    │
│  │      [uptrace-dsn]      │   │  processors:                 │    │
│  │    ─── tenant-aware ─── │   │    batch/system (10K / 10s)  │    │
│  │                         │   │    resourcedetection          │    │
│  │  exporter:              │   │                              │    │
│  │    otlp/clients         │   │  exporters:                  │    │
│  │    (headers_setter ext) │   │    otlp/system               │    │
│  │    forwards uptrace-dsn │   │    prometheusremotewrite     │    │
│  │    from request context │   │    (static DSN header)       │    │
│  └────────────┬────────────┘   └──────────────┬───────────────┘    │
│               │                               │                     │
└───────────────┼───────────────────────────────┼─────────────────────┘
                │                               │
                ▼                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Uptrace  (2.1.0-beta)                         │
│                                                                     │
│         UI · Alerting · Service Graph · Sourcemaps                  │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  ClickHouse  │  │  PostgreSQL  │  │    Redis     │              │
│  │  25.8.15.35  │  │  17.9-alpine │  │  8.6.1-alpine│              │
│  │              │  │              │  │              │              │
│  │  traces      │  │  users       │  │  cache       │              │
│  │  logs        │  │  orgs        │  │  sessions    │              │
│  │  metrics     │  │  projects    │  │              │              │
│  │              │  │  dashboards  │  │              │              │
│  │  ZSTD(1)     │  │  alerts      │  │              │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
└─────────────────────────────────────────────────────────────────────┘
```
---
## Key Design Decisions
### 1. Dual-Pipeline Isolation
The OTel Collector runs **two completely separate pipelines**:

| Aspect              | Client Pipeline                                                                                                                                                                           | System Pipeline                                        |
|---------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| **Purpose**         | App telemetry from your services                                                                                                                                                          | Infrastructure self-monitoring                         |
| **Auth mechanism**  | `headers_setter/clients` extension — forwards `uptrace-dsn` from incoming request metadata                                                                                                | Static `uptrace-dsn` header injected at exporter level |
| **Why separate?**   | System scrapers (`hostmetrics`, `postgresql`, `redis`) generate telemetry internally — there is no incoming HTTP/gRPC request carrying a DSN token. They **cannot** use `headers_setter`. |
| **Batch processor** | `batch/clients` with `metadata_keys: [uptrace-dsn]`                                                                                                                                       | `batch/system` (no metadata keys)                      |
| **Pipelines**       | `traces`, `logs`, `metrics/clients`                                                                                                                                                       | `metrics/system-otlp`, `metrics/system-prometheus`     |

### 2. Tenant-Aware Batching
`batch/clients` declares `metadata_keys: [uptrace-dsn]`, which means the collector creates **separate batches per unique DSN value**. This prevents cross-tenant data mixing when multiple projects send telemetry through the same collector.
### 3. Fail-Fast Environment Validation
All **16 environment variables** use Docker Compose's `${VAR:?}` syntax. If any variable is missing or empty, `docker compose up` **refuses to start** — no silent failures, no half-running stacks.
### 4. Security by Default
- PostgreSQL port is **not exposed** to the host — only accessible within the Docker network
- Redis, ClickHouse — internal only
- Only the OTel Collector OTLP ports (gRPC + HTTP) and Uptrace UI are exposed
---
## Tech Stack

| Component          | Image                                          | Role                                                              |
|--------------------|------------------------------------------------|-------------------------------------------------------------------|
| **Uptrace**        | `uptrace/uptrace:2.1.0-beta`                   | Observability UI, alerting, service graph, sourcemaps             |
| **OTel Collector** | `otel/opentelemetry-collector-contrib:0.123.0` | Telemetry ingestion, routing, batching (6 receivers, 5 pipelines) |
| **ClickHouse**     | `clickhouse/clickhouse-server:25.8.15.35`      | Columnar storage for traces, logs, metrics (ZSTD(1) compression)  |
| **PostgreSQL**     | `postgres:17.9-alpine`                         | Metadata storage (users, orgs, projects, dashboards, alerts)      |
| **Redis**          | `redis:8.6.1-alpine`                           | In-memory cache, session storage                                  |
---

## Project Structure
```
.
├── docker-compose.yml      # 5 services, 16 env vars, healthchecks on all
├── otel-collector.yaml     # Collector config: 6 receivers, 3 processors, 3 exporters, 5 pipelines
├── uptrace.yml             # Uptrace server config: ClickHouse schema, storage policies, seed data
├── .env                    # Your secrets (not committed — see .env.example below)
└── README.md
```
---
## Quick Start
### 1. Clone
```bash
git clone https://github.com/thumbrise/uptrace-template-basic.git
cd uptrace-template-basic
```
### 2. Create `.env`
```bash
cat > .env << 'EOF'
# Uptrace
UPTRACE_SECRET=your-secret-key-change-me
UPTRACE_HTTP=14318
UPTRACE_GRPC=14317
UPTRACE_ADMIN_EMAIL=admin@example.com
UPTRACE_ADMIN_PASSWORD=changeme
UPTRACE_SYSTEM_PROJECT_TOKEN=system_project_token_changeme
# ClickHouse
CLICKHOUSE_USER=uptrace
CLICKHOUSE_PASSWORD=uptrace
CLICKHOUSE_DATABASE=uptrace
# PostgreSQL
POSTGRESQL_USER=uptrace
POSTGRESQL_PASSWORD=uptrace
POSTGRESQL_DATABASE=uptrace
# Redis
REDIS_USERNAME=default
REDIS_PASSWORD=redispass
# OTel Collector ports
OTLP_COLLECTOR_GRPC=4317
OTLP_COLLECTOR_HTTP=4318
EOF
```
### 3. Start
```bash
docker compose up -d
```
All 5 services have healthchecks. The OTel Collector waits for Uptrace to be healthy before starting.
### 4. Open Uptrace UI
```
http://localhost:14318
```
Login with the email and password from your `.env`.
### 5. Send Telemetry
Point any OpenTelemetry SDK at the collector:

| Protocol | Endpoint         |
|----------|------------------|
| **gRPC** | `localhost:4317` |
| **HTTP** | `localhost:4318` |

Set the `uptrace-dsn` header (or metadata key) to your project DSN:
```
http://<project_token>@localhost:14318?grpc=14317
```
> **⚠️ Important:** The header key is `uptrace-dsn`, not `X-OTLP-Token` (which was used in older versions).
---

## Integrating Your App (Copy-Paste)
The OpenTelemetry SDK in **any language** can be configured entirely via environment variables — zero code changes needed. Add these to your app's `.env`, `docker-compose.yml`, Kubernetes manifest, or whatever you use:
### Universal env block (works for every OTel SDK)
```bash
# ── OpenTelemetry SDK configuration ──────────────────────────────────
OTEL_SERVICE_NAME=my-service
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
OTEL_EXPORTER_OTLP_HEADERS=uptrace-dsn=http://<PROJECT_TOKEN>@localhost:14318?grpc=14317
OTEL_RESOURCE_ATTRIBUTES=service.version=1.0.0,deployment.environment=production
```
> Replace `<PROJECT_TOKEN>` with the token from Uptrace UI → Project → Settings → DSN.
> If your app runs **inside the same Docker network**, replace `localhost` with service names:
> `OTEL_EXPORTER_OTLP_ENDPOINT=http://otelcol:4317` and DSN host with `uptrace`.
### Docker Compose example (app in the same stack)
```yaml
services:
  my-app:
    image: my-app:latest
    environment:
      OTEL_SERVICE_NAME: my-app
      OTEL_EXPORTER_OTLP_PROTOCOL: grpc
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otelcol:4317
      OTEL_EXPORTER_OTLP_HEADERS: "uptrace-dsn=http://<PROJECT_TOKEN>@uptrace:14318?grpc=14317"
    depends_on:
      otelcol:
        condition: service_healthy
```
### HTTP instead of gRPC
```bash
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
```
### How it flows
```
Your app (OTel SDK)
  │
  │  OTEL_EXPORTER_OTLP_HEADERS sets the uptrace-dsn metadata
  │  on every gRPC/HTTP request to the collector
  │
  ▼
OTel Collector (receivers.otlp, include_metadata: true)
  │
  │  batch/clients reads uptrace-dsn from metadata
  │  → batches per-tenant (metadata_keys: [uptrace-dsn])
  │
  │  headers_setter/clients forwards uptrace-dsn to Uptrace
  │
  ▼
Uptrace (routes to the correct project by DSN token)
```
### Quick reference: all OTEL_* env vars you might need
| Variable                      | Required | Example                        | What it does                            |
|-------------------------------|----------|--------------------------------|-----------------------------------------|
| `OTEL_SERVICE_NAME`           | ✅        | `my-service`                   | Service name in traces/metrics          |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | ✅        | `http://localhost:4317`        | Collector address                       |
| `OTEL_EXPORTER_OTLP_HEADERS`  | ✅        | `uptrace-dsn=http://token@...` | Auth header — routes to Uptrace project |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | ✅        | `grpc` or `http/protobuf`      | Transport protocol                      |
| `OTEL_RESOURCE_ATTRIBUTES`    | —        | `service.version=1.0.0,...`    | Extra resource attributes               |
| `OTEL_TRACES_SAMPLER`         | —        | `parentbased_traceidratio`     | Sampling strategy                       |
| `OTEL_TRACES_SAMPLER_ARG`     | —        | `0.1`                          | Sample 10% of traces                    |
| `OTEL_LOGS_EXPORTER`          | —        | `otlp`                         | Enable log export (some SDKs)           |
| `OTEL_METRICS_EXPORTER`       | —        | `otlp`                         | Enable metrics export (some SDKs)       |
> **That's it.** No Uptrace SDK, no vendor lock-in. Standard OpenTelemetry env vars → standard OTLP protocol → this collector → Uptrace.
---

## OTel Collector Pipelines at a Glance
```
traces:              otlp ──► batch/clients ──► otlp/clients
logs:                otlp ──► batch/clients ──► otlp/clients
metrics/clients:     otlp ──► batch/clients ──► otlp/clients
metrics/system-otlp: hostmetrics ─┐
                     postgresql ──┤► batch/system + resourcedetection ──► otlp/system
                     httpcheck ───┤
                     redis ───────┘
metrics/system-prom: prometheus/self ──► batch/system ──► prometheusremotewrite/system
```
---
## ClickHouse Tuning
| Parameter            | Value                                     | Notes                            |
|----------------------|-------------------------------------------|----------------------------------|
| Compression          | `ZSTD(1)`                                 | Good ratio, moderate CPU         |
| `dial_timeout`       | `3s`                                      | Connection establishment         |
| `write_timeout`      | `5s`                                      | Write operations                 |
| `max_retries`        | `3`                                       | Failed operation retries         |
| `max_execution_time` | `15s`                                     | Query timeout                    |
| Storage policies     | 8 × `default`                             | Pre-wired for SSD/HDD/S3 tiering |
| Cluster mode         | `replicated: false`, `distributed: false` | Knobs ready for scaling          |
---
## Environment Variables Reference
| Variable                       | Used By                      | Description                      |
|--------------------------------|------------------------------|----------------------------------|
| `UPTRACE_SECRET`               | uptrace                      | Server secret key                |
| `UPTRACE_HTTP`                 | uptrace, otelcol             | HTTP listen port                 |
| `UPTRACE_GRPC`                 | uptrace, otelcol             | gRPC listen port                 |
| `UPTRACE_ADMIN_EMAIL`          | uptrace                      | Super admin email (seed data)    |
| `UPTRACE_ADMIN_PASSWORD`       | uptrace                      | Super admin password (seed data) |
| `UPTRACE_SYSTEM_PROJECT_TOKEN` | uptrace, otelcol             | System project DSN token         |
| `CLICKHOUSE_USER`              | uptrace, clickhouse          | ClickHouse credentials           |
| `CLICKHOUSE_PASSWORD`          | uptrace, clickhouse          | ClickHouse credentials           |
| `CLICKHOUSE_DATABASE`          | uptrace, clickhouse          | ClickHouse database name         |
| `POSTGRESQL_USER`              | uptrace, otelcol, postgresql | PostgreSQL credentials           |
| `POSTGRESQL_PASSWORD`          | uptrace, otelcol, postgresql | PostgreSQL credentials           |
| `POSTGRESQL_DATABASE`          | uptrace, otelcol, postgresql | PostgreSQL database name         |
| `REDIS_USERNAME`               | uptrace                      | Redis ACL username               |
| `REDIS_PASSWORD`               | uptrace, otelcol             | Redis password                   |
| `OTLP_COLLECTOR_GRPC`          | otelcol                      | Collector gRPC port              |
| `OTLP_COLLECTOR_HTTP`          | otelcol                      | Collector HTTP port              |
---
## License
This is a template repository. Use it as a starting point for your observability infrastructure.