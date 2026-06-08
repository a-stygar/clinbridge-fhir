# clinbridge-fhir

## Призначення

ClinBridge — навчальний healthcare-interoperability application для побудови
integration layer між нестандартними вхідними даними та downstream FHIR R4 server.

## Вимоги

- Python 3.12+
- Docker Engine
- Docker Compose plugin

Поточне локальне середовище перевірене на Python 3.14.5.

## Створення virtual environment:
    python3 -m venv .venv
    source .venv/bin/activate

    python -m pip install --upgrade pip
    python -m pip install -e ".[dev]"

команда встановлення dependencies:

## Copy .env.example -> .env:
    cp .env.example .env

## Запуск інфраструктури

```bash
docker compose config
docker compose up -d
docker compose ps
```

## Запуск FastAPI

```bash
uvicorn app.main:app --reload
```

## Health і metadata smoke checks:
    ```bash
    python -m pytest
    ruff check .
    ruff format --check .
    python -m pip check
    ```
    ```bash
    curl -fsS http://localhost:8000/health
    ```
    ```bash
    curl -fsS \
    -H "Accept: application/fhir+json" \
    http://localhost:8080/fhir/metadata
    ```
    ```bash
    docker compose logs -f hapi-fhir
    ```

## Зафіксовані container images

- PostgreSQL: `postgres:17` — major-version pin; фактичний patch version перевіряється через
  `SELECT version()`.
- HAPI FHIR JPA Server: `hapiproject/hapi:v8.10.0-1`.
- FHIR version:  `4.0.1`,

> **Local development only:** HAPI працює без authentication і призначений лише для synthetic data.
> Ця конфігурація не є production deployment.
