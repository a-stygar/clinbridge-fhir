# FHIR HTTP Observations

| Case | Method / endpoint | Status | Content-Type | Response type | Important evidence | Conclusion | Limitation |
|---|---|---:|---|---|---|---|---|
| Server metadata | GET /metadata | ... | ... | CapabilityStatement | ... | ... | ... |
| Create Patient | POST /Patient | 201 | ... | Patient | id=1003, Location, ETag, X-Request-ID | ... | ... |
| Read Patient | GET /Patient/1000 | 200 | ... | Patient | id=1000, ETag, Content-Location | ... | ... |
| Missing Patient | GET /Patient/99999 | 404 | ... | OperationOutcome | severity=error, code=processing, diagnostics=... | ... | ... |
| Wrong resource type | POST Observation to /Patient | 400 | ... | OperationOutcome | severity=error, code=processing, diagnostics=... | ... | ... |