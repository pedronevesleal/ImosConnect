# ImosConnect

## Goal
Create an integration layer that pulls production data recorded by the CAD/CAM system from Microsoft SQL Server, forwards it to a cloud API, and presents the synchronized information through a dashboard.

## High-Level Architecture
1. **Data Agent (on-premises)**
   - Reads new/changed rows from the CAD/CAM SQL Server tables.
   - Transforms data into normalized payloads.
   - Pushes batched payloads to the cloud API via HTTPS.
   - Handles retries, offline buffering, and per-record acknowledgements.
2. **Ingestion & API Service (cloud)**
   - Verifies payload signatures/keys and writes data into the operational store (PostgreSQL or SQL Server in Azure).
   - Emits events to a message bus (e.g., Azure Service Bus, Kafka) for downstream processing.
   - Exposes REST/GraphQL endpoints for dashboard consumption.
3. **Dashboard (web)**
   - React/Next.js SPA backed by the API service.
   - Provides live status, historical analytics, and alerting hooks.

```
CAD/CAM SQL Server --> On-prem Agent --> HTTPS --> Cloud API --> DB/Event Bus --> Dashboard
```

## Component Breakdown
### On-Prem Data Agent
- **Language:** Python (FastAPI + SQLAlchemy) or .NET worker.
- **Connectivity:** Uses SQL Server Change Tracking or timestamp columns to fetch deltas.
- **Resilience:** Local LiteDB/SQLite queue for temporary storage when offline.
- **Security:** Mutual TLS + signed JWT with short-lived credentials.
- **Config:** YAML/ENV for table mappings, polling intervals, endpoint URLs.

### Cloud API Service
- **Framework:** FastAPI / .NET Minimal APIs / NestJS.
- **Endpoints:**
  - `POST /v1/payloads`: bulk insert, idempotent via UUID batch ids.
  - `GET /v1/jobs/:id`: agent heartbeat & job configs.
  - `GET /v1/metrics`: aggregated stats for dashboard widgets.
- **Processing:**
  - Validate schema against Pydantic/JSON Schema.
  - Persist to `production_events`, `machines`, `jobs`, `operators` tables.
  - Publish normalized events to queue for analytics pipeline.
- **Observability:** Structured logging, OpenTelemetry traces, Prometheus metrics.

### Dashboard
- **Stack:** Next.js + Tailwind + Recharts (or Plotly) for charts.
- **Features:**
  - Live machine/job status cards.
  - Timeline of recent operations.
  - Filtering by date, machine, operator, job type.
  - Alert settings (threshold breaches trigger webhook/email).
- **Auth:** Azure AD / Auth0 for role-based access.

## Data Model (example tables)
| Table | Key Fields | Notes |
| --- | --- | --- |
| `machines` | `machine_id`, `name`, `status` | Current state/metadata.
| `jobs` | `job_id`, `cad_reference`, `scheduled_start`, `status` | Links to CAD entities.
| `production_events` | `event_id`, `machine_id`, `job_id`, `event_type`, `payload`, `occurred_at` | Append-only stream from agent.
| `operators` | `operator_id`, `name`, `shift` | Optional HR linkage.

## Deployment Strategy
- Agent packaged as Docker container or Windows Service (if machines run Windows).
- API + DB deployed to Azure App Service + Azure SQL/PG, or AWS ECS + RDS.
- Dashboard hosted as static site (Vercel/S3) pulling data from API over HTTPS.
- CI/CD via GitHub Actions running tests, linting, and container builds.

## Next Steps
1. Define exact CAD/CAM tables and required fields.
2. Prototype agent polling + delta tracking against staging SQL Server.
3. Scaffold FastAPI service with auth + payload validation.
4. Stand up dashboard skeleton consuming mock API data.
5. Add monitoring (Azure App Insights, Grafana) and alerting workflows.
