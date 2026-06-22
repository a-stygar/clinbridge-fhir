# Day 3 HTTP Evidence

## Environment

- FHIR base URL: `http://localhost:8080/fhir`
- Server: HAPI FHIR 8.10.0 REST Server
- FHIR version: R4 / 4.0.1
- Data: synthetic only

## Important Dataset Note

The local HAPI server already contained a previously created matching Patient before this evidence batch. Because of that, some searches return more than one result.

This is useful Day 3 evidence: base FHIR search can return duplicate business identifiers, and ClinBridge must not resolve ambiguity by selecting the first entry.

## Create Patient

Request:

```http
POST http://localhost:8080/fhir/Patient
Content-Type: application/fhir+json
Accept: application/fhir+json
```

Request body:

```text
examples/fhir/patient-identifiers.json
```

Response:

```text
HTTP status: 201 Created
Content-Type: application/fhir+json;charset=UTF-8
X-Request-ID: b6UENNBiwjga01T
Location: http://localhost:8080/fhir/Patient/1052/_history/1
Content-Location: http://localhost:8080/fhir/Patient/1052/_history/1
ETag: W/"1"
```

Body summary:

```text
resourceType: Patient
id: 1052
meta.versionId: 1
identifier:
- https://clinic.example.it/id/codice-fiscale | SYNTHETIC-CF-0001
- https://hospital.example.it/id/mrn | MRN-0001
```

Conclusion:

```text
HAPI accepted the synthetic Patient and assigned server Resource.id = 1052.
Resource.id is server-local technical identity and is different from business Identifier.
```

## Create Collision Patient

Request:

```http
POST http://localhost:8080/fhir/Patient
Content-Type: application/fhir+json
Accept: application/fhir+json
```

Request body:

```text
examples/fhir/patient-identifier-collision.json
```

Response:

```text
HTTP status: 201 Created
Content-Type: application/fhir+json;charset=UTF-8
X-Request-ID: ahar6GqFSLobTBTB
Location: http://localhost:8080/fhir/Patient/1053/_history/1
Content-Location: http://localhost:8080/fhir/Patient/1053/_history/1
ETag: W/"1"
Last-Modified: Mon, 22 Jun 2026 10:49:31 GMT
```

Body summary:

```text
resourceType: Patient
id: 1053
meta.versionId: 1
meta.lastUpdated: 2026-06-22T10:49:31.151+00:00
identifier:
- https://other-hospital.example.it/id/mrn | MRN-0001
```

Conclusion:

```text
HAPI accepted a second synthetic Patient with the same visible MRN value under a different identifier system.
This demonstrates namespace separation:
https://hospital.example.it/id/mrn | MRN-0001
is not the same identifier pair as
https://other-hospital.example.it/id/mrn | MRN-0001.
```

## Create Practitioner

Request:

```http
POST http://localhost:8080/fhir/Practitioner
Content-Type: application/fhir+json
Accept: application/fhir+json
```

Request body:

```text
examples/fhir/practitioner.json
```

Response:

```text
HTTP status: 201 Created
Content-Type: application/fhir+json;charset=UTF-8
X-Request-ID: 3h5ff4Shj8LmtXYW
Location: http://localhost:8080/fhir/Practitioner/1054/_history/1
Content-Location: http://localhost:8080/fhir/Practitioner/1054/_history/1
ETag: W/"1"
Last-Modified: Mon, 22 Jun 2026 10:49:58 GMT
```

Body summary:

```text
resourceType: Practitioner
id: 1054
meta.versionId: 1
meta.lastUpdated: 2026-06-22T10:49:58.256+00:00
identifier:
- https://clinic.example.it/id/practitioner-license | PRACT-001
name:
- family: Bianca
- given: Chiara
```

Conclusion:

```text
HAPI accepted the synthetic Practitioner and assigned server Resource.id = 1054.
The Practitioner carries a professional identifier, but no organization or role context is inferred here.
```

## Create Organization

Request:

```http
POST http://localhost:8080/fhir/Organization
Content-Type: application/fhir+json
Accept: application/fhir+json
```

Request body:

```text
examples/fhir/organization.json
```

Response:

```text
HTTP status: 201 Created
Content-Type: application/fhir+json;charset=UTF-8
X-Request-ID: di7ca6pzjFVibwMa
Location: http://localhost:8080/fhir/Organization/1055/_history/1
Content-Location: http://localhost:8080/fhir/Organization/1055/_history/1
ETag: W/"1"
Last-Modified: Mon, 22 Jun 2026 10:50:18 GMT
```

Body summary:

```text
resourceType: Organization
id: 1055
meta.versionId: 1
meta.lastUpdated: 2026-06-22T10:50:18.025+00:00
identifier:
- https://clinic.example.it/id/organization | HOSP-0001
active: true
name: Other Hospital Example
```

Conclusion:

```text
HAPI accepted the synthetic Organization and assigned server Resource.id = 1055.
The organization identifier is the deterministic lookup key; the name is human-readable text and is not a stable identity key.
```

## Search Patient By Value Only

Request:

```http
GET http://localhost:8080/fhir/Patient?identifier=MRN-0001
Accept: application/fhir+json
```

Response:

```text
HTTP status: 200 OK
Content-Type: application/fhir+json;charset=UTF-8
X-Request-ID: 0PAxK2Ssg6h74UAa
resourceType: Bundle
Bundle.type: searchset
Bundle.total: 3
entry count observed: 3
```

Matched entries:

| Resource type | Resource id | Matched identifier evidence |
|---|---:|---|
| Patient | 1051 | `https://hospital.example.it/id/mrn \| MRN-0001` |
| Patient | 1052 | `https://hospital.example.it/id/mrn \| MRN-0001` |
| Patient | 1053 | `https://other-hospital.example.it/id/mrn \| MRN-0001` |

Conclusion:

```text
Value-only search is broader than governed system|value lookup.
It returned matches from multiple identifier systems and also revealed duplicate resources under the same hospital MRN pair.
ClinBridge must not use value-only search for automatic patient linking.
```

## Search Patient By Correct System And Value

Request:

```http
GET http://localhost:8080/fhir/Patient?identifier=https%3A%2F%2Fhospital.example.it%2Fid%2Fmrn%7CMRN-0001
Accept: application/fhir+json
```

Logical searched pair:

```text
https://hospital.example.it/id/mrn | MRN-0001
```

Response:

```text
HTTP status: 200 OK
Content-Type: application/fhir+json;charset=UTF-8
X-Request-ID: sxg0LywUyshZzv34
resourceType: Bundle
Bundle.type: searchset
Bundle.total: 2
entry count observed: 2
```

Matched entries:

| Resource type | Resource id | Matched identifier evidence |
|---|---:|---|
| Patient | 1051 | `https://hospital.example.it/id/mrn \| MRN-0001` |
| Patient | 1052 | `https://hospital.example.it/id/mrn \| MRN-0001` |

Conclusion:

```text
The correct system|value search narrowed the result set compared with value-only search, but it still returned two matches.
This is an ambiguous identity state.
ClinBridge must not select the first result automatically.
Base FHIR server behavior does not guarantee global uniqueness of business identifiers.
```

## Search Patient By Other Hospital System And Same Visible Value

Request:

```http
GET http://localhost:8080/fhir/Patient?identifier=https%3A%2F%2Fother-hospital.example.it%2Fid%2Fmrn%7CMRN-0001
Accept: application/fhir+json
```

Logical searched pair:

```text
https://other-hospital.example.it/id/mrn | MRN-0001
```

Response:

```text
HTTP status: 200 OK
Content-Type: application/fhir+json;charset=UTF-8
X-Request-ID: p0JS0bxRbKL4q7Ap
resourceType: Bundle
Bundle.type: searchset
Bundle.total: 1
entry count observed: 1
```

Matched entries:

| Resource type | Resource id | Matched identifier evidence |
|---|---:|---|
| Patient | 1053 | `https://other-hospital.example.it/id/mrn \| MRN-0001` |

Conclusion:

```text
The same visible value, MRN-0001, resolved to a different Patient when searched under a different identifier system.
This supports the Day 3 rule that Identifier.system + Identifier.value is the working identity pair.
```

## Search Patient By Non-Existing System And Value

Request:

```http
GET http://localhost:8080/fhir/Patient?identifier=https%3A%2F%2Fnon-existing-hospital.example.it%2Fid%2Fmrn%7CMRN-99999
Accept: application/fhir+json
```

Logical searched pair:

```text
https://non-existing-hospital.example.it/id/mrn | MRN-99999
```

Response:

```text
HTTP status: 200 OK
Content-Type: application/fhir+json;charset=UTF-8
X-Request-ID: DmVSWSOw45wUPJBA
resourceType: Bundle
Bundle.type: searchset
Bundle.total: 0
entry presence: absent
Last-Modified: Mon, 22 Jun 2026 10:55:46 GMT
```

Conclusion:

```text
Zero matches is a valid FHIR search outcome.
The server returned HTTP 200 with a searchset Bundle and total = 0, not an error.
ClinBridge must handle this according to workflow policy: create, reject, or escalate later.
```

## Search Practitioner By Professional Identifier

Request:

```http
GET http://localhost:8080/fhir/Practitioner?identifier=https%3A%2F%2Fclinic.example.it%2Fid%2Fpractitioner-license%7CPRACT-001
Accept: application/fhir+json
```

Logical searched pair:

```text
https://clinic.example.it/id/practitioner-license | PRACT-001
```

Response:

```text
HTTP status: 200 OK
Content-Type: application/fhir+json;charset=UTF-8
X-Request-ID: FyxwIyN3hHiHadRi
resourceType: Bundle
Bundle.type: searchset
Bundle.total: 1
entry count observed: 1
```

Matched entries:

| Resource type | Resource id | Matched identifier evidence |
|---|---:|---|
| Practitioner | 1054 | `https://clinic.example.it/id/practitioner-license \| PRACT-001` |

Conclusion:

```text
The governed professional identifier search returned one Practitioner candidate.
The result supports deterministic lookup for the Practitioner only within this synthetic practitioner-license namespace.
No employment, organization, specialty, or PractitionerRole context is inferred from this identifier.
```

## Search Organization By Organization Identifier

Request:

```http
GET http://localhost:8080/fhir/Organization?identifier=https%3A%2F%2Fclinic.example.it%2Fid%2Forganization%7CHOSP-0001
Accept: application/fhir+json
```

Logical searched pair:

```text
https://clinic.example.it/id/organization | HOSP-0001
```

Response:

```text
HTTP status: 200 OK
Content-Type: application/fhir+json;charset=UTF-8
X-Request-ID: j3lXRbQinmXwzKNV
resourceType: Bundle
Bundle.type: searchset
Bundle.total: 1
entry count observed: 1
```

Matched entries:

| Resource type | Resource id | Matched identifier evidence |
|---|---:|---|
| Organization | 1055 | `https://clinic.example.it/id/organization \| HOSP-0001` |

Conclusion:

```text
The governed organization identifier search returned one Organization candidate.
The organization name remains a human-readable label, not the stable identity key.
```

## Read Back By Server Id

### Read Patient 1052

Request:

```http
GET http://localhost:8080/fhir/Patient/1052
Accept: application/fhir+json
```

Response:

```text
HTTP status: 200 OK
Content-Type: application/fhir+json;charset=UTF-8
X-Request-ID: qpyqJOveq0B8Bkaz
Content-Location: http://localhost:8080/fhir/Patient/1052/_history/1
ETag: W/"1"
Last-Modified: Mon, 22 Jun 2026 10:45:38 GMT
```

Body summary:

```text
resourceType: Patient
id: 1052
meta.versionId: 1
meta.lastUpdated: 2026-06-22T10:45:38.048+00:00
identifier:
- https://clinic.example.it/id/codice-fiscale | SYNTHETIC-CF-0001
- https://hospital.example.it/id/mrn | MRN-0001
```

### Read Patient 1053

Request:

```http
GET http://localhost:8080/fhir/Patient/1053
Accept: application/fhir+json
```

Response:

```text
HTTP status: 200 OK
Content-Type: application/fhir+json;charset=UTF-8
X-Request-ID: Z1KAmGEvMYMY9ph2
Content-Location: http://localhost:8080/fhir/Patient/1053/_history/1
ETag: W/"1"
Last-Modified: Mon, 22 Jun 2026 10:49:31 GMT
```

Body summary:

```text
resourceType: Patient
id: 1053
meta.versionId: 1
meta.lastUpdated: 2026-06-22T10:49:31.151+00:00
identifier:
- https://other-hospital.example.it/id/mrn | MRN-0001
```

### Read Practitioner 1054

Request:

```http
GET http://localhost:8080/fhir/Practitioner/1054
Accept: application/fhir+json
```

Response:

```text
HTTP status: 200 OK
Content-Type: application/fhir+json;charset=UTF-8
X-Request-ID: 0GgOvsLnQGN5Nc3w
Content-Location: http://localhost:8080/fhir/Practitioner/1054/_history/1
ETag: W/"1"
Last-Modified: Mon, 22 Jun 2026 10:49:58 GMT
```

Body summary:

```text
resourceType: Practitioner
id: 1054
meta.versionId: 1
meta.lastUpdated: 2026-06-22T10:49:58.256+00:00
identifier:
- https://clinic.example.it/id/practitioner-license | PRACT-001
name:
- family: Bianca
- given: Chiara
```

### Read Organization 1055

Request:

```http
GET http://localhost:8080/fhir/Organization/1055
Accept: application/fhir+json
```

Response:

```text
HTTP status: 200 OK
Content-Type: application/fhir+json;charset=UTF-8
X-Request-ID: HxijKfvO2FZVwNRW
Content-Location: http://localhost:8080/fhir/Organization/1055/_history/1
ETag: W/"1"
Last-Modified: Mon, 22 Jun 2026 10:50:18 GMT
```

Body summary:

```text
resourceType: Organization
id: 1055
meta.versionId: 1
meta.lastUpdated: 2026-06-22T10:50:18.025+00:00
identifier:
- https://clinic.example.it/id/organization | HOSP-0001
active: true
name: Other Hospital Example
```

Conclusion:

```text
Read-back by server-assigned ids succeeded for Patient/1052, Patient/1053, Practitioner/1054, and Organization/1055.
The read-back responses confirm that Resource.id is server-local technical identity and that business identity remains represented by Identifier.system + Identifier.value.
```

## Zero / One / Multiple Result Policy

### Zero Matches

Zero matches means no known resource was found for the searched identifier pair. It is a valid search outcome, not a transport failure. ClinBridge may create, reject, or escalate according to a later workflow contract.

Observed example:

```text
https://non-existing-hospital.example.it/id/mrn | MRN-99999
returned Bundle.total = 0 and no entry.
```

### One Match

One match means one candidate resource was found for the searched identifier pair. ClinBridge may treat it as a deterministic candidate only after verifying the resource type and expected identifier pair.

Observed example:

```text
https://other-hospital.example.it/id/mrn | MRN-0001
returned Patient/1053 only.
https://clinic.example.it/id/practitioner-license | PRACT-001
returned Practitioner/1054 only.
https://clinic.example.it/id/organization | HOSP-0001
returned Organization/1055 only.
```

### Multiple Matches

Multiple matches means ambiguous identity state.

Observed examples:

```text
identifier=MRN-0001 returned 3 Patient matches.
identifier=https://hospital.example.it/id/mrn|MRN-0001 returned 2 Patient matches.
```

Conclusion:

```text
ClinBridge must not select the first result automatically.
Ambiguous identity state requires stop, escalation, cleanup, or governed reconciliation.
```

## Overall Day 3 Evidence Conclusion

The lab demonstrates that:

- `Identifier.system + Identifier.value` is the working identity pair.
- `Identifier.value` alone is ambiguous.
- Equal-looking values in different systems are not automatically equal identities.
- A FHIR server may contain duplicate business identifiers unless stronger profile, server, or workflow rules prevent them.
- `Resource.id` is server-local technical identity and does not replace business identifiers.
- Deterministic identifier lookup is not an MPI, demographic matcher, or merge authority.

Day 3 HTTP evidence is complete for the required create, read-back, and identifier search cases:

- POST Patient, collision Patient, Practitioner, and Organization;
- read-back by server-assigned ids;
- value-only Patient search;
- correct Patient system|value search;
- other-hospital Patient system|value search;
- non-existing Patient system|value search;
- Practitioner system|value search;
- Organization system|value search.
