# clinbridge-fhir

призначення проєкту:
ClinBridge - FHIR intergation application, який приймає довільні дані (JSON) віl upstream
    клієнта, перетворює іх у FHIR формат, дадсилає дані у HAPI серер, який зберігає дані у HAPI PostgreSL DB

вимоги до Python:
    Python 3.12+

створення virtual environment:
    python3.14 -m venv .venv

команда встановлення dependencies:

копіювання .env.example -> .env:
    .env не комітиться

docker compose up -d;

health і metadata smoke checks:
    uvicorn app.main:app --reload
    pytest
    ruff check .
    ruff format --check .
    pytest
    docker compose logs -f hapi-fhir

PostgreSQL v17
HAPI v8.10.0-1

пояснення topology:
    приймає довільні дані (JSON) віl upstream
    клієнта, перетворює іх у FHIR формат, дадсилає дані у HAPI серер, який зберігає дані у HAPI PostgreSL DB

!середовище локальне й без authentication!