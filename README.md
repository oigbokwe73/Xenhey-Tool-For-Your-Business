# Xenhey-Tool-For-Your-Business
**Xenhey** is a **workflow + process-driven integration tool** that ships a **runtime DLL via NuGet** and can connect to Azure services (Service Bus, Event Hubs, Azure SQL, ADX, Storage, etc.), while configuraiton of all rules uses **REST operations** for AI readiness.

---

## What Xenhey is (in plain English)

**Xenhey is an integration runtime + workflow engine packaged as a NuGet-deployed DLL** that you drop into your .NET service (API, Worker, Function, etc.). Once installed, it provides:

* A **standard way to connect** to multiple Azure resources (messaging, data, storage, analytics)
* A **workflow/process layer** that orchestrates steps like “ingest → validate → enrich → route → persist → report”
* A consistent **REST interface** for running workflows and retrieving operational/analytics reporting (everything is “a REST call”)

Think of it as:
**“A plug-in integration runtime that turns Azure resources into reusable workflow steps—without rewriting custom connectors every time.”**

---

## Core building block: the NuGet runtime DLL

### Why a NuGet runtime matters

Shipping Xenhey as a NuGet package means:

* **Fast adoption**: dev teams add one dependency instead of pulling multiple SDKs + writing glue code
* **Consistent patterns**: same retry policies, logging, error handling, telemetry, auth
* **Versioned capability**: connector improvements roll out like normal package upgrades
* **Host flexibility**: can run inside:

  * ASP.NET Web API
  * Azure Functions (isolated worker)
  * Worker Service
  * Container Apps / AKS microservices

### What the runtime “does” at startup

The Xenhey runtime typically:

1. Loads configuration (JSON/YAML/appsettings/Key Vault)
2. Registers connectors (Service Bus, Event Hub, SQL, ADX, Storage, Azure Document Search, All Openapi systems etc.)
3. Wires cross-cutting concerns:

   * Managed Identity / service principal auth
   * retries + backoff
   * circuit breakers
   * structured logging + correlation IDs
   * metrics + tracing (App Insights / OpenTelemetry)
4. Exposes workflow endpoints via REST

---

## The workflow/process-driven model

Xenhey is not “just a connector library.” The key is the **workflow engine**:

### Workflow = a chain of steps

A workflow is a named process made of steps such as:

* **Trigger step**

  * HTTP request
  * Service Bus subscription message
  * Event Hub batch
  * Blob-created event
* **Transform step**

  * mapping fields
  * JSON normalization
  * schema enforcement
* **Route step**

  * based on rules (x-api-key-based routing)
  * push to Service Bus topic, Event Hub, SQL, ADX, Storage
  * The ability to extent via new conectors to more workflows
* **Persist step**

  * SQL insert/merge
  * ADX ingestion
  * store blob/file + metadata
* **Report step**

  * produce workflow run summary
  * expose the output via REST

### Why “process-driven” matters

This design makes Xenhey good for repeatable enterprise workflows:

* onboarding pipelines
* ingestion pipelines
* event-to-data lake patterns
* “fan-out” to multiple sinks (SQL + ADX + Storage)
* auditing and traceability across steps

---

## Connectors Xenhey supports (conceptually)

### Azure Service Bus (queue/topic/subscription)

Xenhey connector typically supports:

* Send message (with headers, correlation, scheduled delivery)
* Receive/peek/lock/complete/abandon/dead-letter
* Topic publish with routing properties
* Retry + DLQ handling baked into workflow policies

**Use case:** orchestrate business workflows reliably (exactly-once-ish patterns, idempotency, compensations).

---

### Azure Event Hubs

Xenhey connector typically supports:

* Publish events in batches
* Consume events with checkpoints
* Attach metadata (partition key, event properties)
* Stream-oriented ingestion into ADX or other stores

**Use case:** telemetry-style ingestion, high-throughput streaming into analytics.

---

### Azure SQL Database

Xenhey connector typically supports:

* Parameterized queries
* bulk insert/merge patterns
* stored procedure execution
* transactional steps within a workflow
* idempotency keys to avoid duplicate writes

**Use case:** operational persistence, reporting tables, integration state.

---

### Azure Data Explorer (ADX / Kusto)

Xenhey connector typically supports:

* ingest JSON/CSV/parquet
* KQL query execution
* “workflow telemetry” tables (e.g., runs/errors/latency)
* time-series analytics reporting

**Use case:** near-real-time analytics + powerful KQL reporting.

---

### Azure Storage (Blob / Table / Queue / Files)

Xenhey connector typically supports:

* write blobs + metadata tags
* read blobs for processing
* table storage for lightweight state + audit records
* queue storage for simple buffering (if used)

**Use case:** durable payload storage, replay, audit trail, cheap state store.

---

## “All reporting is REST calls” — what that really means

Xenhey’s philosophy is typically:

1. **Every workflow run** produces structured run data (start/end, steps, success/failure, durations)
2. That data is persisted (SQL/ADX/Storage Table) in a standardized format
3. Xenhey exposes reporting through REST endpoints like:

* `GET /api/workflows` → list workflows
* `POST /api/workflows/{name}/run` → trigger a workflow
* `GET /api/runs?workflow=name&from=&to=` → list runs
* `GET /api/runs/{runId}` → full run details (step-by-step)
* `GET /api/runs/{runId}/logs` → correlated logs
* `GET /api/metrics?workflow=name` → aggregates (success rate, latency, throughput)
* `GET /api/resources/health` → connection checks to SB/EH/SQL/ADX/Storage

That means consumers (Power BI, dashboards, admin portals, SRE tooling) don’t need direct DB/Kusto access—**they just call the API**.

---

## Operational value (why this is better than “custom glue code”)

### Standardization

* One consistent integration pattern across many Azure services
* Same authentication method (Managed Identity strongly preferred)
* Same error model (validation vs transient vs fatal)
* Same retry/backoff strategy

### Governance + security

* Centralized policies:

  * allowed resources
  * approved topics/hubs/tables
  * RBAC-controlled access
* Secrets handled properly (Key Vault, MI) instead of config sprawl
* Clear audit trails: who ran what workflow, what data moved where

### Observability (SRE-friendly)

* Correlation ID from REST request → Service Bus message → SQL/ADX writes
* Unified telemetry: step duration, connector latency, failure reasons
* Easy “run replay” if blob payload + run metadata are retained

---

## Typical runtime architecture (high-level)

**Host app (your API or Function)**

* references Xenhey NuGet runtime DLL
* reads config
* Xenhey registers connectors
* Xenhey loads workflow definitions
* exposes REST endpoints
* writes run history + metrics to SQL/ADX
* logs to App Insights

**Your consumer UI / Power BI / admin portal**

* never calls Azure resources directly
* calls Xenhey REST endpoints for:

  * run workflows
  * query run history
  * fetch metrics / reports

---


### Xenhey Workflow – Rules-Driven, API-Secured

```mermaid
flowchart TD
    A[Client / System / UI] -->|REST Call<br/>x-api-key| B[Xenhey REST API]

    B --> C[API Gateway Layer<br/>Auth + Throttling]
    C -->|Validate x-api-key| D[Workflow Runtime Engine]

    D --> E[Load Workflow Definition]
    E --> F[Load Rules JSON<br/>from Config / Storage / Key Vault]

    F --> G[Rule Evaluation Engine]
    G -->|Apply Routing & Validation Rules| H{Decision Point}

    %% Messaging paths
    H -->|Async Workflow| I[Azure Service Bus<br/>Queue / Topic]
    H -->|Streaming Events| J[Azure Event Hub]

    %% Data persistence paths
    H -->|Transactional Data| K[Azure SQL Database]
    H -->|Analytics / Time-Series| L[Azure Data Explorer (ADX)]
    H -->|Payload / Audit Storage| M[Azure Storage<br/>Blob / Table]

    %% Execution tracking
    I --> N[Workflow Step Execution]
    J --> N
    K --> N
    L --> N
    M --> N

    %% Observability
    N --> O[Telemetry & Logging]
    O --> P[Application Insights / OpenTelemetry]

    %% Reporting
    N --> Q[Persist Run Metadata<br/>Status, Duration, Errors]
    Q --> R[Reporting API Layer]

    R -->|REST Response| S[Admin UI / Power BI / Consumers]

    %% Error handling
    N -->|Failure| T[Retry / Dead-Letter / Compensation]
    T --> Q
```

---

## What this diagram communicates (talk track)

### 1️⃣ Secure API entry

* All workflow execution and reporting is exposed via **REST**
* Every call is authenticated using an **`x-api-key`**
* Optional throttling, IP allowlists, or RBAC can sit at the API layer

---

### 2️⃣ Rules are JSON-driven

* Workflow behavior is **not hardcoded**
* Rules are defined in **JSON** (routing, validation, thresholds, destinations)
* Rules can be stored in:

  * Azure Storage
  * Azure SQL
  * Azure App Config
  * Azure Key Vault (secured)

Example rule intent:

* *“If eventType = Billing → send to Service Bus + SQL”*
* *“If telemetry → Event Hub + ADX”*
* *“If payload > 5MB → Blob Storage”*

---

### 3️⃣ Workflow runtime orchestration

* Xenhey runtime evaluates rules
* Determines:

  * which Azure service to call
  * sync vs async execution
  * retry and failure behavior
* Same runtime, different workflows — purely config-driven

---

### 4️⃣ Multi-target Azure integration

From the same workflow execution:

* **Service Bus** → business workflows
* **Event Hub** → high-throughput streams
* **Azure SQL** → operational data
* **ADX** → analytics and observability
* **Storage** → raw payloads and audit trails

---

### 5️⃣ Built-in observability and reporting

* Every step emits:

  * execution time
  * success/failure
  * correlation ID
* Telemetry flows to App Insights / OpenTelemetry
* Run metadata is persisted and exposed via **REST reporting APIs**
* No direct DB or Kusto access needed for consumers

---

### 6️⃣ Failure handling is first-class

* Retries, DLQs, and compensations are workflow-aware
* Failures are still reported via REST
* Enables replay and forensic analysis

---

