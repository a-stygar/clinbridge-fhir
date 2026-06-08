Upstream source
      |
      v
ClinBridge
      |
      | FHIR HTTP API
      v
HAPI FHIR Server
      |
      | JDBC / SQL — HAPI-owned internal boundary
      v
HAPI PostgreSQL


## Ownership

### ClinBridge owns

- inbound integration workflow;
- raw source payloads later;
- canonical normalization and mapping decisions later;
- idempotency, outbox, retry, replay and operational audit later.

### HAPI owns

- FHIR REST interactions;
- FHIR resource persistence;
- resource versions and history;
- FHIR search indexes;
- HAPI internal database schema.

## Prohibited coupling

- ClinBridge must not query or modify HAPI internal tables.
- ClinBridge must not run Alembic migrations against the HAPI database.
- HAPI must not use the ClinBridge application schema.
- The supported integration boundary is the FHIR HTTP API.
