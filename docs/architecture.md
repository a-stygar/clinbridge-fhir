Developer/test client
  -> ClinBridge FastAPI
       -> ClinBridge PostgreSQL

Developer/test client
  -> HAPI FHIR
       -> HAPI PostgreSQL

ClinBridge працює з HAPI тільки через FHIR HTTP API.
ClinBridge не читає HAPI internal tables.
HAPI не використовує ClinBridge schema.
