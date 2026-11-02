# Technical Design Document (TDD)

---

## 1. Document Control & Architecture Metadata

| Attribute | Details |
| :--- | :--- |
| **Project / Service Name** | `[Service / System Name]` |
| **PRD Reference** | `[Link to PRD v1.0.0]` |
| **Status** | `[Draft | In Review | Approved | Deprecated]` |
| **Author(s)** | `[Lead Engineer / Architect]` |
| **Reviewer(s)** | `[Engineering Manager / Security / DevOps]` |
| **Target Implementation Sprint** | `[Sprint Number or Date]` |

---

## 2. System Overview & Architecture Diagram

### 2.1 High-Level Architecture
Provide a high-level summary of the components and network topology. Describe how client applications communicate with backend services, queues, and persistent storage.

```
┌─────────────────────────────────────────────────────────────┐
│                       Client Layers                         │
│  [ Web App (Next.js) ]           [ Marketing Site (Astro) ] │
└──────────────┬──────────────────────────────┬───────────────┘
               │ HTTPS                        │ HTTPS
               ▼                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway / Edge Router                │
│  - Cloudflare / Reverse Proxy (SSL, DDoS, Rate Limiting)    │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                    Application Core Layer                   │
│  [ Primary REST / tRPC API ]       [ Webhook Receivers ]    │
└──────────────┬──────────────────────────────┬───────────────┘
               │                              │
       ┌───────┴────────┐             ┌───────┴────────┐
       ▼                ▼             ▼                ▼
┌─────────────┐  ┌─────────────┐ ┌─────────────┐  ┌─────────────┐
│  PostgreSQL │  │ Redis Cache │ │ BullMQ / SQS│  │ Third-Party │
│  (Database) │  │  & Sessions │ │  Worker Job │  │   (Stripe)  │
└─────────────┘  └─────────────┘ └─────────────┘ └─────────────┘
```

### 2.2 Technology Stack Justification
| Layer | Technology | Decision Rational | Trade-offs & Alternatives Considered |
| :--- | :--- | :--- | :--- |
| **Frontend** | Next.js (App Router), Tailwind | Server-side rendering, component ecosystem, fast DX | Slower build times compared to Vite SPA |
| **Backend API** | Node.js (TypeScript) / Express | Shared types with frontend, high async I/O throughput | Higher CPU usage for heavy compute vs. Go |
| **Database** | PostgreSQL (Amazon RDS / Supabase) | ACID compliance, strong relational integrity, JSONB support | Scalability requires read replicas vs. DynamoDB |
| **Caching / Queue** | Redis + BullMQ | Fast in-memory atomic operations, robust job retries | In-memory limits require eviction strategy |

---

## 3. Data Storage & Schema Design

### 3.1 Entity-Relationship Model
All primary entities must include audit timestamps (`created_at`, `updated_at`) and multi-tenant partition keys (`org_id`).

```
┌───────────────────────────┐         1:N         ┌───────────────────────────┐
│       organizations       │────────────────────<│           users           │
├───────────────────────────┤                     ├───────────────────────────┤
│ id: uuid (PK)             │                     │ id: uuid (PK)             │
│ name: varchar(255)        │                     │ org_id: uuid (FK)         │
│ slug: varchar(64) UNIQUE  │                     │ email: varchar(255) UNIQUE│
│ stripe_customer_id: text  │                     │ role: user_role (ENUM)    │
│ plan_tier: varchar(32)    │                     │ created_at: timestamptz   │
│ created_at: timestamptz   │                     └───────────────────────────┘
└─────────────┬─────────────┘
              │
              │ 1:N
              ▼
┌───────────────────────────┐         1:N         ┌───────────────────────────┐
│        workspaces         │────────────────────<│         resources         │
├───────────────────────────┤                     ├───────────────────────────┤
│ id: uuid (PK)             │                     │ id: uuid (PK)             │
│ org_id: uuid (FK)         │                     │ workspace_id: uuid (FK)   │
│ name: varchar(128)        │                     │ org_id: uuid (FK)         │
│ created_at: timestamptz   │                     │ title: varchar(255)       │
└───────────────────────────┘                     │ status: resource_status   │
                                                  │ metadata: jsonb           │
                                                  │ deleted_at: timestamptz   │
                                                  └───────────────────────────┘
```

### 3.2 Database Indexes & Partitioning Strategy
```sql
-- Enforce tenant isolation indexes for fast multi-tenant queries
CREATE INDEX idx_users_org_id ON users(org_id);
CREATE INDEX idx_resources_workspace_lookup ON resources(workspace_id, org_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_resources_metadata_gin ON resources USING gin (metadata);
```

### 3.3 Data Migration & Seeding Strategy
* Migrations run sequentially via toolchain (e.g., Prisma Migrate / Drizzle Kit / Flyway).
* Zero-downtime policy: Destructive operations (drop column, rename) require a two-phase rollout (Expand and Contract pattern).

---

## 4. API Interface & Data Contracts

All endpoints return uniform response envelopes and use standardized HTTP status codes.

### 4.1 Response Envelope Format
```json
{
  "success": true,
  "data": {},
  "error": null,
  "meta": {
    "requestId": "req_01HPX7K...",
    "timestamp": "2026-11-02T17:00:00Z"
  }
}
```

### 4.2 Endpoint Specifications

#### `POST /api/v1/workspaces/:workspaceId/resources`
Creates a new primary resource within the target workspace.

* **Authentication:** Bearer JWT (Session Header)
* **Authorization:** Requires role `Admin` or `Editor` in target tenant.
* **Rate Limit:** 30 requests / minute per user token.

**Request Payload:**
```json
{
  "title": "Quarterly Financial Analysis",
  "metadata": {
    "tags": ["finance", "q3"],
    "priority": "high"
  }
}
```

**Response Payload (`201 Created`):**
```json
{
  "success": true,
  "data": {
    "id": "res_98a72b14-49c0",
    "workspaceId": "ws_110283fa",
    "title": "Quarterly Financial Analysis",
    "status": "pending_processing",
    "createdAt": "2026-11-02T17:00:00Z"
  },
  "error": null,
  "meta": {
    "requestId": "req_881273aa",
    "timestamp": "2026-11-02T17:00:00Z"
  }
}
```

**Error Responses:**
* `400 Bad Request`: Payload validation failed (Zod schema rejection).
* `401 Unauthorized`: Missing or expired session token.
* `403 Forbidden`: User does not hold rights in `org_id`.
* `429 Too Many Requests`: Rate limit exceeded.

---

## 5. Third-Party Integrations & Webhook Ingestion

### 5.1 Stripe Billing Lifecycle
All billing mutations depend on webhook event synchronization, never on client-side success redirects.

```
Client                Stripe Server               Our Webhook Handler           Postgres DB
  │                         │                              │                         │
  │─── Checkout Session ───>│                              │                         │
  │                         │─── checkout.session.comp ───>│                         │
  │                         │                              │── Check Idempotency ───>│
  │                         │                              │── Update Org Tier ─────>│
  │                         │<─── HTTP 200 OK ─────────────│                         │
```

* **Idempotency Strategy:** Store all incoming `event.id` values in a `processed_webhooks` table. Reject duplicates immediately with `HTTP 200 OK`.
* **Signature Verification:** Validate the raw request body with Stripe's signing secret before executing payload deserialization.

---

## 6. Security, Authentication & Isolation Architecture

* **Multi-Tenancy Isolation:**
  * Application level: Middleware verifies that target resource `org_id` matches the session claims.
  * Database level: Row Level Security (RLS) configured on tables where applicable:
    ```sql
    ALTER TABLE resources ENABLE ROW LEVEL SECURITY;
    CREATE POLICY tenant_isolation_policy ON resources
      USING (org_id = current_setting('app.current_org_id')::uuid);
    ```
* **Authentication Token Handling:**
  * JWT tokens stored in `httpOnly`, `Secure`, `SameSite=Lax` cookies to prevent XSS exfiltration.
  * Passwords hashed using `argon2id` (minimum memory cost: 64MB).
* **Secret Management:**
  * No secrets committed to source control. Variables injected at runtime via secret managers (AWS Secrets Manager, Doppler, or Infisical).

---

## 7. Performance, Caching & Scalability

* **Caching Strategy:**
  * Read-heavy queries (e.g., workspace settings, profile states) cached in Redis with a 15-minute Time-To-Live (TTL).
  * Cache invalidation executed explicitly on `PATCH`, `PUT`, or `DELETE` triggers for the given resource key (`tenant:{org_id}:workspaces`).
* **Background Worker Concurrency:**
  * Long-running processing jobs offloaded to Redis BullMQ workers.
  * Worker concurrency capped at 10 jobs per instance with exponential backoff retries ($1\text{s}, 5\text{s}, 25\text{s}$).

---

## 8. Observability, Telemetry & Incident Runbook

### 8.1 Logging, Metrics & Tracing
* **Structured Logging:** All logs output formatted JSON stdout including `level`, `message`, `requestId`, `userId`, `orgId`, and `latencyMs`.
* **Error Tracking:** Unhandled exceptions captured in Sentry with contextual breadcrumbs and stack traces.

### 8.2 Health Checks & SLAs
* Health check endpoint `/api/healthz` verifies:
  1. Primary PostgreSQL read/write connectivity.
  2. Redis ping response $< 10\text{ms}$.
  3. Disk storage threshold $< 85\%$.

### 8.3 Incident Rollback Procedure
1. Identify failure threshold (e.g., HTTP 5xx error rate $> 1\%$ for $\ge 3$ consecutive minutes).
2. Redeploy previous immutable image/container tag via deployment pipeline.
3. If database migration occurred, execute backwards-compatible down-migration script:
   ```bash
   pnpm db:migrate:rollback
   ```
4. Verify traffic health on telemetry dashboards.