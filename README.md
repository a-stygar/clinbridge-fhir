# clinbridge-fhir

uvicorn app.main:app --reload
pytest
ruff check .
ruff format --check .

docker compose logs -f hapi-fhir