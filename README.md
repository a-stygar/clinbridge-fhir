# ClinBridge FHIR

ClinBridge is a healthcare interoperability learning and portfolio project.

The application is being built as an integration layer between non-FHIR healthcare inputs and a downstream FHIR R4 server. Its long-term responsibilities include preserving source payloads, normalizing them into an internal canonical model, mapping them deterministically to FHIR R4, validating the result, delivering it reliably, and recording processing evidence.

ClinBridge is **not** a custom FHIR server. The local environment uses HAPI FHIR as a separate downstream FHIR repository and API.

## Current status

The project is currently in its initial infrastructure slice.

Implemented:

- FastAPI application skeleton;
- `GET /health`;
- pytest smoke test;
- Ruff configuration;
- local ClinBridge PostgreSQL service;
- separate PostgreSQL service for HAPI FHIR;
- HAPI FHIR JPA Server configured in R4 mode;
- architecture and terminology documentation.

Not implemented yet:

- referral ingestion;
- canonical healthcare models;
- FHIR resource mapping;
- FHIR validation pipeline;
- transaction Bundles;
- idempotency and outbox delivery;
- retry, dead-letter, and replay;
- OAuth2 or SMART Backend Services;
- HL7v2 or Mirth integration.

The repository contains only synthetic development data. It is not a production healthcare deployment.

## Architecture

```text
Developer / test client
        |
        +--> ClinBridge FastAPI :8000
        |       |
        |       +--> ClinBridge PostgreSQL :5432
        |
        +--> HAPI FHIR R4 :8080/fhir
                |
                +--> HAPI PostgreSQL
```

ClinBridge and HAPI have separate persistence ownership:

```text
ClinBridge database
  Future ClinBridge-owned raw payloads, workflow state,
  idempotency records, outbox entries, and operational audit.

HAPI database
  HAPI-owned FHIR resources, versions, indexes, and internal JPA schema.
```

ClinBridge may communicate with HAPI only through the FHIR HTTP API. It must not read or modify HAPI's internal database tables.

See [docs/architecture.md](docs/architecture.md) for the complete boundary description.

## Technology baseline

| Component | Current baseline |
|---|---|
| Python | `>=3.12` |
| Web framework | FastAPI |
| ASGI server | Uvicorn |
| Tests | pytest |
| Linting and formatting | Ruff |
| ClinBridge database | PostgreSQL 17 |
| HAPI database | PostgreSQL 17 |
| FHIR server | HAPI FHIR JPA Server |
| HAPI image | `hapiproject/hapi:v8.10.0-1` |
| FHIR family | R4 |
| Project FHIR baseline | FHIR R4 4.0.1 |
| Local orchestration | Docker Compose |

The PostgreSQL images are pinned to the major version `17`, not to an exact patch version or image digest. The actual runtime patch version must therefore be recorded from the running containers.

## Prerequisites

Required locally:

- Git;
- Python 3.12 or newer;
- Docker Engine or Docker Desktop;
- Docker Compose plugin;
- `curl`, or an equivalent HTTP client.

Check the main tools:

```bash
git --version
python --version
docker --version
docker compose version
```

## Clone the repository

```bash
git clone https://github.com/a-stygar/clinbridge-fhir.git
cd clinbridge-fhir
```

Confirm that the shell and Git point to the intended clone:

```bash
pwd
git rev-parse --show-toplevel
git rev-parse HEAD
git status --short
```

In PowerShell, use `Get-Location` instead of `pwd` when preferred.

## Create the Python environment

### Linux or macOS

```bash
python3 -m venv .venv
source .venv/bin/activate

python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
python -m pip check
```

### Windows PowerShell

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1

python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
python -m pip check
```

The project currently declares dependency ranges in `pyproject.toml`, but no Python lock file is committed. Exact transitive dependency reproduction is therefore not yet guaranteed.

## Configure local environment variables

Copy the example file.

### Linux or macOS

```bash
cp .env.example .env
```

### Windows PowerShell

```powershell
Copy-Item .env.example .env
```

The committed values are local-development examples only:

```dotenv
APP_ENV=local
APP_NAME=ClinBridge

CLINBRIDGE_DATABASE_URL=postgresql+psycopg://clinbridge:clinbridge@localhost:5432/clinbridge

FHIR_BASE_URL=http://localhost:8080/fhir

CLINBRIDGE_DB_NAME=clinbridge
CLINBRIDGE_DB_USER=clinbridge
CLINBRIDGE_DB_PASSWORD=clinbridge

HAPI_DB_NAME=hapi
HAPI_DB_USER=hapi
HAPI_DB_PASSWORD=hapi
```

Do not place production credentials, tokens, certificates, or real patient data in `.env` or any committed file.

## Start the infrastructure

Validate the resolved Compose configuration:

```bash
docker compose config
```

Start the three services:

```bash
docker compose up -d
docker compose ps
```

Expected services:

```text
clinbridge-db
hapi-db
hapi-fhir
```

Follow HAPI startup logs when necessary:

```bash
docker compose logs -f hapi-fhir
```

The containers expose:

| Service | Host endpoint | Purpose |
|---|---|---|
| `clinbridge-db` | `localhost:5432` | ClinBridge-owned PostgreSQL |
| `hapi-db` | `localhost:5433` | Diagnostic host access to HAPI PostgreSQL |
| `hapi-fhir` | `http://localhost:8080/fhir` | Downstream FHIR API |

Inside the Compose network, HAPI connects to its database through:

```text
jdbc:postgresql://hapi-db:5432/hapi
```

It does not connect through the host port `5433`.

## Run ClinBridge

With the virtual environment activated:

```bash
python -m uvicorn app.main:app --reload
```

The application is available at:

```text
http://localhost:8000
```

FastAPI documentation is available at:

```text
http://localhost:8000/docs
```

## Verify ClinBridge health

```bash
curl -fsS http://localhost:8000/health
```

Expected response:

```json
{
  "status": "ok"
}
```

This endpoint currently proves only that the ClinBridge process can serve an HTTP request. It does not verify PostgreSQL or HAPI readiness.

PowerShell alternative:

```powershell
Invoke-RestMethod http://localhost:8000/health
```

## Verify HAPI FHIR metadata

```bash
curl -fsS \
  -H "Accept: application/fhir+json" \
  http://localhost:8080/fhir/metadata
```

Inspect at least:

```text
resourceType = CapabilityStatement
fhirVersion
software.name and software.version, when present
implementation description, when present
supported formats
rest.mode
advertised resource types
advertised interactions
```

A successful `/metadata` response proves that the server advertises a FHIR capability surface. It does not prove that every advertised interaction behaves correctly for the project use case.

## Verify database identity and isolation

ClinBridge database:

```bash
docker compose exec clinbridge-db \
  psql -U clinbridge -d clinbridge \
  -c "SELECT version(), current_database(), current_user;"
```

HAPI database:

```bash
docker compose exec hapi-db \
  psql -U hapi -d hapi \
  -c "SELECT version(), current_database(), current_user;"
```

The outputs must demonstrate different database names and users:

```text
ClinBridge:
  database = clinbridge
  user     = clinbridge

HAPI:
  database = hapi
  user     = hapi
```

Inspect schemas:

```bash
docker compose exec clinbridge-db \
  psql -U clinbridge -d clinbridge \
  -c "\dn"

docker compose exec hapi-db \
  psql -U hapi -d hapi \
  -c "\dn"
```

After HAPI has started successfully, its database should contain HAPI-created objects. ClinBridge must not create, migrate, query, or modify those objects.

Inspect mounted storage:

```bash
docker inspect clinbridge_postgres \
  --format '{{range .Mounts}}{{println .Name "->" .Destination}}{{end}}'

docker inspect hapi_postgres \
  --format '{{range .Mounts}}{{println .Name "->" .Destination}}{{end}}'
```

The containers must use different named volumes.

## Record runtime versions

Configured values:

```text
PostgreSQL image: postgres:17
HAPI image:       hapiproject/hapi:v8.10.0-1
HAPI mode:        R4
```

Record observed runtime evidence rather than relying only on image labels.

PostgreSQL:

```bash
docker compose exec clinbridge-db \
  psql -U clinbridge -d clinbridge \
  -c "SELECT version();"
```

HAPI image:

```bash
docker inspect hapi_fhir --format '{{.Config.Image}}'
```

FHIR version and HAPI software information must be taken from the actual `CapabilityStatement`.

## Controlled dependency failure

Stop HAPI:

```bash
docker compose stop hapi-fhir
```

Run the metadata request again:

```bash
curl -fsS \
  -H "Accept: application/fhir+json" \
  http://localhost:8080/fhir/metadata
```

The command must fail clearly. It must not produce a false success result.

Restore HAPI:

```bash
docker compose start hapi-fhir
docker compose ps
```

This test distinguishes application liveness from downstream dependency availability. It is not a retry implementation.

## Run quality checks

From the repository root:

```bash
python -m pytest
ruff check .
ruff format --check .
python -m pip check
```

To confirm pytest is collecting tests from the intended clone:

```bash
python -m pytest --collect-only
```

Inspect the reported `rootdir` and collected test path.

## Stop the local stack

Preserve data:

```bash
docker compose down
```

Delete containers and local named volumes:

```bash
docker compose down -v
```

The `-v` form permanently deletes local PostgreSQL data for both services.

## Project structure

```text
clinbridge-fhir/
├── app/
│   ├── api/
│   │   └── routes/
│   │       └── health.py
│   ├── core/
│   ├── db/
│   ├── tests/
│   │   └── test_health.py
│   └── main.py
├── docs/
│   ├── architecture.md
│   └── glossary.md
├── requests/
├── .env.example
├── .gitignore
├── docker-compose.yml
├── pyproject.toml
└── README.md
```

## Important boundaries

ClinBridge owns, or will own:

- inbound integration workflows;
- preservation of source payloads;
- canonical normalization;
- deterministic mapping decisions;
- validation orchestration;
- delivery state;
- idempotency and outbox records;
- retry and replay decisions;
- operational audit.

HAPI owns:

- FHIR REST interactions;
- FHIR resource persistence;
- resource version history;
- FHIR search indexes;
- HAPI's internal JPA schema;
- its advertised `CapabilityStatement`.

Prohibited coupling:

- ClinBridge must not write directly to the HAPI database;
- ClinBridge must not query HAPI internal tables;
- ClinBridge must not apply Alembic migrations to the HAPI database;
- HAPI must not use ClinBridge application tables;
- cross-database joins are not a supported integration contract.

## Local security limitations

The current local HAPI server:

- has no authentication;
- has no authorization policy;
- has no TLS termination;
- uses simple development credentials;
- is intended only for synthetic data;
- is not a production service.

A reachable local HAPI server is not evidence of SMART Backend Services conformance or production security.

## Documentation

- [Architecture and ownership boundaries](docs/architecture.md)
- [FHIR and integration glossary](docs/glossary.md)

## Scope discipline

The initial infrastructure slice deliberately excludes:

- Keycloak;
- OAuth2 and SMART;
- Patient mapping;
- canonical referral models;
- transaction Bundles;
- FHIR profile authoring;
- terminology packages;
- outbox, retry, and dead-letter processing;
- Mirth or HL7v2;
- AWS deployment.
