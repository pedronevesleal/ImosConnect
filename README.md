# ImosConnect

## Goal
Create an integration layer that pulls production data recorded by CAD/CAM SQL Server instances hosted on customer sites, forwards it to a multi-tenant cloud API, and presents the synchronized information through an authenticated dashboard.

## High-Level Architecture
1. **Data Agent (on-premises)**
   - Customer-specific deployment (Windows Service or container) that reads new/changed rows from local CAD/CAM SQL Server tables.
   - Includes a bootstrap wizard/CLI to capture customer tenant ID, API credentials, polling intervals, and SQL connection strings.
   - Transforms data into normalized payloads.
   - Pushes batched payloads to the cloud API via HTTPS using tenant-scoped API keys or client certificates.
   - Handles retries, offline buffering, and per-record acknowledgements.
2. **Ingestion & API Service (cloud)**
   - Verifies payload signatures/keys and writes data into the operational store (PostgreSQL or SQL Server in Azure).
   - Emits events to a message bus (e.g., Azure Service Bus, Kafka) for downstream processing.
   - Exposes REST/GraphQL endpoints for dashboard consumption.
3. **Dashboard (web)**
   - React/Next.js SPA backed by the API service.
   - Provides live status, historical analytics, and alerting hooks.
   - Enforces tenant isolation by deriving access scopes from the identity provider (customer users only see their own data).

```
CAD/CAM SQL Server --> On-prem Agent --> HTTPS --> Cloud API --> DB/Event Bus --> Dashboard
```

## Component Breakdown
### On-Prem Data Agent
- **Language:** Python (FastAPI + SQLAlchemy) or .NET worker.
- **Connectivity:** Uses SQL Server Change Tracking or timestamp columns to fetch deltas.
- **Resilience:** Local LiteDB/SQLite queue for temporary storage when offline.
- **Security:** Mutual TLS + signed JWT with short-lived credentials; tenant-scoped API key rotated per customer.
- **Config:** YAML/ENV for table mappings, polling intervals, endpoint URLs, tenant ID, client secret, SQL credentials.
- **Provisioning Flow:** IT admin runs `imos-agent setup` which:
  1. Prompts for CAD/CAM SQL connection info and tests connectivity.
  2. Retrieves tenant config from the cloud API using a one-time enrollment code.
  3. Stores encrypted credentials in OS key store and writes operational config to disk.
  4. Registers host fingerprint so the control plane can monitor heartbeats.

- **Identity/Access:** Every API call is tied to a tenant ID resolved from the presented credential. Middleware enforces tenant-based row-level security.
- **Framework:** FastAPI / .NET Minimal APIs / NestJS.
- **Endpoints:**
  - `POST /v1/payloads`: bulk insert, idempotent via UUID batch ids, requires tenant-signed JWT.
  - `GET /v1/jobs/:id`: agent heartbeat & job configs filtered per tenant.
  - `GET /v1/metrics`: aggregated stats for dashboard widgets, scoped per tenant.
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
- **Auth:** Azure AD / Auth0 multi-tenant configuration; login determines tenant context, roles (admin/operator/viewer) and permissible datasets.
- **Tenant Isolation:** GraphQL/REST queries automatically include tenant ID claims so that UI cannot mix or leak data across customers.

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

## Tenant Provisioning & Access Control
- Maintain a control-plane service (or admin UI) where new customers are registered, triggering:
  - Creation of tenant record with quotas and encryption keys.
  - Issuance of one-time enrollment codes for the on-prem agent.
  - Assignment of default dashboard roles (e.g., admin, supervisor).
- Each customer’s data lives in shared tables with a `tenant_id` column and row-level security policies.
- Observability dashboards and alert channels are filtered per tenant to avoid cross-contamination.

## Next Steps
1. Define exact CAD/CAM tables and required fields.
2. Prototype agent polling + delta tracking against staging SQL Server.
3. Implement tenant onboarding service + enrollment code issuance.
4. Scaffold FastAPI service with auth + payload validation.
5. Stand up dashboard skeleton consuming mock API data w/ tenant scopes.
6. Add monitoring (Azure App Insights, Grafana) and alerting workflows.
