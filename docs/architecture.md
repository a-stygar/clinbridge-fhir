# ClinBridge Architecture

## 1. Purpose

ClinBridge is a healthcare integration application that will receive non-FHIR healthcare input, preserve the original evidence, normalize the input into a domain-specific canonical model, map it deterministically to FHIR R4, validate the output, and deliver it to a downstream FHIR server.

ClinBridge is not a FHIR repository implementation. In the local environment, HAPI FHIR is treated as an external downstream system with its own API, data model, persistence lifecycle, and schema ownership.

The project starts as a modular monolith. This keeps the integration pipeline explicit and avoids introducing distributed-system complexity before it is justified.

## 2. Current Day 0 topology

```text
Developer / test client
        |
        | HTTP :8000
        v
ClinBridge FastAPI
        |
        | GET /health
        v
Process-local health response


Developer / test client
        |
        | PostgreSQL protocol :5432
        v
ClinBridge PostgreSQL
        |
        v
clinbridge_postgres_data


Developer / test client
        |
        | HTTP :8080/fhir
        v
HAPI FHIR JPA Server
        |
        | JDBC inside Docker network
        | jdbc:postgresql://hapi-db:5432/hapi
        v
HAPI PostgreSQL
        |
        v
hapi_postgres_data
```

At this stage:

- ClinBridge exposes only the health endpoint;
- ClinBridge persistence models are not implemented yet;
- the ClinBridge PostgreSQL service establishes the future application-owned persistence boundary;
- HAPI exposes the downstream FHIR API;
- HAPI persists its internal state in its own PostgreSQL service;
- the test client calls ClinBridge and HAPI separately for smoke verification;
- a ClinBridge outbound FHIR client has not yet been implemented.

The documentation must not imply that a data pipeline exists before the corresponding code is present.

## 3. Planned integration flow

```text
Upstream healthcare source
        |
        | custom JSON initially
        | HL7v2 through Mirth later
        v
ClinBridge inbound adapter
        |
        v
Raw source payload preservation
        |
        v
Domain-specific canonical referral model
        |
        v
Deterministic canonical-to-FHIR mapping
        |
        v
FHIR R4 validation
        |
        v
Reliable delivery boundary
        |
        | FHIR HTTP API
        v
Downstream HAPI FHIR server
        |
        v
HAPI-owned FHIR persistence
```

Later delivery reliability will include local idempotency, outbox processing, bounded retry, dead-letter handling, replay controls, and operational audit. Those components are planned responsibilities, not current Day 0 capabilities.

## 4. Runtime components

### 4.1 ClinBridge FastAPI

Current responsibilities:

- application composition;
- HTTP process boundary;
- `GET /health`;
- future inbound API endpoints.

Future responsibilities:

- source request acceptance;
- configuration and dependency wiring;
- canonical validation;
- mapping orchestration;
- downstream FHIR client coordination;
- structured error responses;
- operational endpoints.

ClinBridge routes should remain thin. Business and interoperability decisions should not accumulate inside FastAPI route functions.

### 4.2 ClinBridge PostgreSQL

Current purpose:

- reserve and verify a separate application-owned persistence boundary;
- prove local connectivity and volume isolation.

Future data may include:

- raw source messages;
- source identity and correlation identifiers;
- canonical processing state;
- idempotency or inbox records;
- outbox entries;
- delivery attempts;
- dead-letter state;
- replay decisions;
- append-only operational audit.

ClinBridge migrations will apply only to the ClinBridge database.

### 4.3 HAPI FHIR JPA Server

HAPI responsibilities include:

- exposing FHIR REST endpoints;
- parsing and processing supported FHIR interactions;
- assigning and managing server-local resource identity;
- persisting FHIR resources and versions;
- maintaining FHIR search indexes;
- advertising supported behavior through a `CapabilityStatement`.

HAPI is a downstream dependency. It is not an internal ClinBridge module.

### 4.4 HAPI PostgreSQL

The HAPI database contains implementation-specific JPA persistence structures owned by HAPI.

Its tables are not:

- ClinBridge domain models;
- a public integration contract;
- a substitute for the FHIR API;
- a supported source for ClinBridge joins or reports;
- a target for ClinBridge Alembic migrations.

The HAPI schema may change when HAPI is upgraded. ClinBridge must therefore remain independent of it.

## 5. Deployment view

| Component | Runtime location | Host exposure | Persistent storage |
|---|---|---|---|
| ClinBridge FastAPI | Local Python process | `localhost:8000` | None yet |
| ClinBridge PostgreSQL | Docker container | `localhost:5432` | `clinbridge_postgres_data` |
| HAPI PostgreSQL | Docker container | `localhost:5433` | `hapi_postgres_data` |
| HAPI FHIR | Docker container | `localhost:8080` | Through HAPI PostgreSQL |

The host port `5433` exists for local diagnostics. HAPI itself connects to `hapi-db:5432` through the Docker Compose network.

The current use of separate PostgreSQL containers is a verification and ownership choice. The essential invariant is logical isolation: separate databases, roles, credentials, schemas, migrations, and ownership.

## 6. System ownership

### 6.1 ClinBridge owns

ClinBridge owns, or will own:

- inbound integration workflow;
- source-system contracts;
- preservation of raw source payloads;
- correlation identifiers;
- canonical normalization rules;
- canonical-to-FHIR mapping rules;
- mapping versions;
- validation orchestration;
- delivery state;
- retry classification;
- replay policy;
- local idempotency state;
- outbox and delivery-attempt records;
- operational audit.

### 6.2 HAPI owns

HAPI owns:

- the FHIR REST implementation;
- its `CapabilityStatement`;
- FHIR resource persistence;
- server-assigned resource ids;
- resource version history;
- search indexes;
- supported server operations;
- its internal JPA database schema;
- HAPI-specific migrations and upgrades.

### 6.3 Upstream sources own

An upstream system owns the meaning and authority of the data it sends, including:

- source message identity;
- source business identifiers;
- source timestamps;
- source terminology;
- the distinction between known, absent, and inferred information.

ClinBridge must not invent clinical meaning that is absent from the source contract.

## 7. Persistence ownership

```text
ClinBridge code
    |
    | SQLAlchemy / migrations later
    v
ClinBridge database only


ClinBridge code
    |
    | FHIR HTTP
    v
HAPI FHIR server
    |
    | HAPI JPA
    v
HAPI database only
```

| Concern | ClinBridge database | HAPI database |
|---|---:|---:|
| Raw inbound payload | Yes, later | No |
| Source message identity | Yes, later | No |
| Idempotency record | Yes, later | No |
| Outbox state | Yes, later | No |
| Delivery attempts | Yes, later | No |
| Local pipeline audit | Yes, later | No |
| FHIR resource persistence | No | Yes |
| FHIR resource history | No | Yes |
| HAPI search indexes | No | Yes |
| HAPI internal schema | No | Yes |

FHIR `Provenance`, optional `AuditEvent`, and ClinBridge operational audit are separate concepts.

## 8. Supported integration boundaries

Allowed boundaries:

```text
Upstream source -> ClinBridge inbound contract
ClinBridge -> ClinBridge database
ClinBridge -> HAPI through FHIR HTTP
HAPI -> HAPI database through HAPI-owned persistence
Developer/test tooling -> local service endpoints for verification
```

The intended downstream boundary is:

```http
Accept: application/fhir+json
Content-Type: application/fhir+json
```

The server's `CapabilityStatement` must be inspected before optional interactions, search parameters, or operations are assumed to exist.

Advertised support is not the same as verified behavior.

## 9. Prohibited coupling

Explicitly prohibited:

- ClinBridge querying HAPI internal tables;
- ClinBridge writing HAPI internal tables;
- ClinBridge applying Alembic migrations to the HAPI database;
- HAPI using ClinBridge application tables;
- cross-database joins as an integration mechanism;
- treating an HAPI table name as a stable public contract;
- bypassing FHIR HTTP processing to create resources through SQL;
- placing HAPI-owned and ClinBridge-owned tables in one shared application schema;
- reusing one database credential for both applications.

Direct database access would bypass supported server behavior, couple ClinBridge to one HAPI version, and make replacement or upgrade unsafe.

## 10. FHIR boundary

FHIR is the downstream interoperability representation. It is not:

- the original source format;
- the raw evidence store;
- the complete internal ClinBridge workflow model;
- a guarantee of clinical correctness;
- a replacement for explicit source contracts.

```text
Raw source representation
        !=
Canonical referral representation
        !=
FHIR output representation
```

## 11. Identity boundary

```text
FHIR Resource.id
  Technical logical identity of a resource on one FHIR server.

FHIR Identifier
  Business identity issued within a named namespace.

Source message id
  Identity of an inbound integration message.

Source referral id
  Business identity of the referral in the source system.

Correlation id
  Technical identifier used to trace one processing flow.

Canonical URL
  Stable identity of a conformance artifact such as a profile.
```

A server-local resource id must not be used as a national identifier, hospital MRN, or source-message id.

Patient matching and merge authority are not owned by the Day 0 system.

## 12. Health and readiness

### Health or liveness

```http
GET /health
```

```json
{
  "status": "ok"
}
```

It answers:

> Can the ClinBridge process serve a basic HTTP request?

It does not prove:

- PostgreSQL connectivity;
- HAPI availability;
- HAPI database availability;
- availability of a required FHIR operation;
- end-to-end readiness.

### Readiness

A future readiness signal may verify required dependencies and configuration.

Readiness must not be added as a large generic subsystem during Day 0. The current negative test checks HAPI availability explicitly through `/metadata`.

## 13. Failure-layer model

```text
Configuration layer
  Missing or invalid settings.

DNS / network layer
  Name resolution, connection refusal, timeout.

HTTP layer
  Status code, headers, content type, raw body.

FHIR layer
  OperationOutcome, unsupported interaction, invalid resource.

Canonical layer
  Invalid normalized application data.

Business-semantic layer
  Source information cannot safely support the requested mapping.

Persistence layer
  ClinBridge database or HAPI-owned persistence failure.
```

A single generic error such as `FHIR error` loses retry and troubleshooting information.

## 14. CapabilityStatement role

```http
GET http://localhost:8080/fhir/metadata
Accept: application/fhir+json
```

A `CapabilityStatement` can advertise:

- FHIR version;
- response formats;
- server software and implementation information;
- REST mode;
- supported resource types;
- resource interactions;
- search parameters;
- operations;
- security declarations.

It is an advertised contract, not a complete behavioral or conformance proof.

## 15. Version boundaries

Configured local baseline:

```text
PostgreSQL image:
  postgres:17

HAPI image:
  hapiproject/hapi:v8.10.0-1

HAPI FHIR mode:
  R4

Project FHIR baseline:
  HL7 FHIR R4 4.0.1
```

The repository must distinguish configured values from observed runtime values.

The exact observed PostgreSQL patch version, HAPI software information, and FHIR version must be recorded after runtime verification.

## 16. Security boundary

The Day 0 environment is intentionally local and unsecured.

Current limitations:

- no HAPI authentication;
- no HAPI authorization policy;
- no TLS termination;
- simple development credentials;
- no secrets manager;
- no production logging policy;
- no backup or disaster-recovery configuration;
- no production network isolation.

Allowed data:

```text
Synthetic data only
```

Not allowed:

```text
Real patient data
Real credentials
Production tokens
Private certificates or keys
Copied PHI-bearing production logs
```

Local HAPI access without authentication is not evidence of OAuth2 or SMART Backend Services conformance.

## 17. Modular-monolith boundary

ClinBridge starts as one deployable application with internal modules:

```text
API and inbound adapters
Configuration
Persistence
Canonical referral domain
FHIR mapping
Validation
Delivery client
Outbox worker
Audit and replay
```

These are internal code ownership boundaries, not independent microservices.

Microservices, Kafka, and Kubernetes are outside the current demonstrated need.

## 18. Planned Mirth boundary

```text
Mirth
  MLLP transport
  Routing
  Basic HL7v2 structure checks
  Minimal extraction and normalization
  ACK/NACK handling
  Message browsing and reprocessing

ClinBridge
  Canonical model ownership
  Semantic validation
  FHIR mapping
  FHIR validation
  Reliable downstream delivery
  Audit and replay
```

Mirth must not become a second independent implementation of the same semantic mapper unless a separate comparison artifact is explicitly intended.

## 19. Deferred concerns

- canonical referral models;
- Patient and ServiceRequest mapping;
- transaction Bundles;
- base FHIR validation subsystem;
- custom profiles;
- terminology services;
- conditional create;
- idempotency;
- outbox and delivery workers;
- dead-letter and replay;
- OAuth2 and SMART;
- Mirth and HL7v2;
- XML, CDA, and SOAP;
- production deployment.

## 20. Architecture decisions summary

| Decision | Reason |
|---|---|
| Start with a modular monolith | Keep one understandable delivery unit and explicit internal boundaries |
| Treat HAPI as an external downstream system | Preserve a standards-based HTTP integration contract |
| Use separate PostgreSQL services locally | Make ownership and isolation visible |
| Give each database separate credentials and storage | Prevent accidental schema coupling |
| Keep `/health` simple | Avoid confusing process health with dependency readiness |
| Pin the HAPI image | Prevent silent upgrades through `latest` |
| Use synthetic data only | The local stack lacks production healthcare security controls |
| Defer Keycloak and SMART | Security mechanics do not belong before the basic FHIR path exists |
| Preserve raw, canonical, and FHIR representations separately later | Support replay, traceability, deterministic mapping, and troubleshooting |
