# FHIR HTTP Observations

This document records observed HTTP and FHIR behavior from the local HAPI FHIR R4 server.

Environment:

```text
FHIR base URL: http://localhost:8080/fhir
FHIR version: 4.0.1 / R4
HAPI FHIR version: 8.10.0
Data: synthetic only
```

## Observed requests and responses

| Case | Method / endpoint | Request Content-Type | Accept | Status | Response Content-Type | Response type | Important evidence | Conclusion | Limitation |
|---|---|---|---|---:|---|---|---|---|---|
| Server metadata | `GET /metadata` | N/A | `application/fhir+json` | `200 OK` | `application/fhir+json;charset=UTF-8` | `CapabilityStatement` | The response declares FHIR version `4.0.1 / R4`, server mode, supported resource types, interactions, search parameters, and operations. | The HAPI FHIR server is reachable and advertises an R4 CapabilityStatement. | A CapabilityStatement is an advertised capability contract. It does not prove that every advertised interaction, operation, search parameter, profile, or validation behavior works correctly. |
| Create Patient | `POST /Patient` | `application/fhir+json` | `application/fhir+json` | `201 Created` | `application/fhir+json;charset=UTF-8` | `Patient` | Returned `Patient.id = 1003`, `identifier.value = PAT-0003`, `meta.versionId = 1`. Response included `Location`, `Content-Location`, `ETag`, `Last-Modified`, and `X-Request-ID`. | HAPI accepted the synthetic Patient payload, created a new Patient resource, assigned a server-local logical id, and returned the created representation. | A successful create does not prove complete profile validation, complete terminology validation, clinical correctness, uniqueness of the business identifier, or idempotent processing. Repeating the same ordinary POST may create another resource. |
| Read Patient | `GET /Patient/1000` | N/A | `application/fhir+json` | `200 OK` | `application/fhir+json;charset=UTF-8` | `Patient` | Returned `resourceType = Patient`, `id = 1000`, `meta.versionId = 1`. Response included `ETag`, `Content-Location`, `Last-Modified`, and `X-Request-ID`. | A Patient resource with logical id `1000` existed on this FHIR server and was readable at the time of the request. | This response does not prove that the resource represents a unique real-world person, that its business identifiers are globally unique, or that equivalent Patient resources do not exist elsewhere. |
| Search Patients | `GET /Patient?_count=10` | N/A | `application/fhir+json` | `200 OK` | `application/fhir+json;charset=UTF-8` | `Bundle` with `type = searchset` | The response contains Patient resources in `Bundle.entry[].resource`. `_count=10` requests a page containing up to ten search results. | FHIR search returns a searchset Bundle rather than a raw JSON list of Patient resources. | `_count=10` does not guarantee that all Patient resources are returned. Additional pages may exist and must be followed through `Bundle.link` with `relation = next`. |
| Missing Patient | `GET /Patient/99999` | N/A | `application/fhir+json` | `404 Not Found` | `application/fhir+json;charset=UTF-8` | `OperationOutcome` | `issue.severity = error`, `issue.code = processing`, diagnostics: `HAPI-2001: Resource Patient/99999 is not known`. Response also included `X-Request-ID`. | The HTTP request reached HAPI successfully, but no Patient resource was returned for logical id `99999`. HAPI provided a structured FHIR explanation through OperationOutcome. | This proves only that this logical resource id was not known to this server at request time. It does not prove that the real-world person does not exist, that no matching Patient exists under another id, or that every `404` from every infrastructure layer will contain an OperationOutcome. |
| Wrong resource type | `POST /Patient` with `resourceType = Observation` | `application/fhir+json` | `application/fhir+json` | `400 Bad Request` | `application/fhir+json;charset=UTF-8` | `OperationOutcome` | `issue.severity = error`, `issue.code = processing`. Diagnostics include `HAPI-0450` and `HAPI-1814`: expected `Patient` but found `Observation`. Response included `X-Request-ID`. | The JSON request reached the FHIR endpoint, but HAPI rejected it because the body resource type did not match the `/Patient` endpoint. This is a deterministic endpoint/resource-type mismatch. | This response does not prove that the Observation itself is invalid when submitted to the correct endpoint. It also does not prove that every FHIR server will return the same status code, diagnostics text, or HAPI-specific error codes. |

## Evidence-preservation rule

For every received HTTP response, preserve evidence in this order:

```text
1. HTTP status code
2. response headers
3. response Content-Type
4. raw response body
5. parsed JSON, when parsing succeeds
6. FHIR resourceType
7. OperationOutcome fields, when the body is actually an OperationOutcome
```

For a transport failure, an HTTP response may not exist. Preserve instead:

```text
requested method and URL
client exception or exit code
transport diagnostics
timeout or connection information
confirmation that status, headers, and response body are absent
```

## Request and response Content-Type distinction

```text
Request Content-Type
  Describes the format of the request body.
  It is required for POST requests that send FHIR JSON.

Accept
  Describes the response format requested by the client.
  It is used for both GET and POST requests.

Response Content-Type
  Describes the actual format returned by the server.
  It may be present for responses to GET, POST, and unsuccessful requests.
```

Example:

```http
GET /Patient/1000
Accept: application/fhir+json
```

The GET request has no body, so request `Content-Type` is not needed. The server response can still contain:

```text
Content-Type: application/fhir+json;charset=UTF-8
```

## Failure-layer classification

```text
Transport failure
  No HTTP response was received.
  Examples: connection refused, DNS failure, TLS failure, timeout.

HTTP failure response
  The server or intermediary returned an HTTP status, headers, and possibly a body.

FHIR error response
  The response body is a valid FHIR resource such as OperationOutcome.

Business or semantic failure
  The request may be structurally valid but violates a source contract,
  identity rule, mapping decision, or workflow policy.
```

Do not collapse these layers into one generic `FHIR error`.

## Retry note

The two unsuccessful responses require different reasoning:

```text
GET /Patient/99999
  The resource was not known to this server at the observed request time.
  Repeating the request immediately without evidence of a state change is unlikely
  to help, but the result is not permanently deterministic because server state may change.

POST Observation to /Patient
  This is a deterministic request defect.
  Repeating the unchanged request cannot correct the endpoint/resource-type mismatch.