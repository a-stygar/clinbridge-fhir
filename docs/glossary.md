# ClinBridge FHIR Glossary

This glossary defines the core terms used in the early ClinBridge slices.

The definitions are practical engineering explanations rather than copied specification text.

## FHIR

**FHIR** stands for **Fast Healthcare Interoperability Resources**.

It is an HL7 interoperability standard that defines reusable healthcare data structures, datatypes, references, conformance artifacts, terminology-related structures, and several exchange patterns.

FHIR is not only:

- a REST API;
- JSON;
- a database schema;
- one vendor product.

REST and JSON are important ways to use FHIR, but the standard also covers XML, operations, messaging, documents, services, profiles, and implementation guidance.

## Resource

A **resource** is a top-level FHIR exchange unit representing a healthcare, administrative, workflow, infrastructure, or conformance concept.

Every resource instance identifies its type through `resourceType`.

```json
{
  "resourceType": "Patient"
}
```

A resource is not the same thing as a database row, Python class, API endpoint, or profile.

## Element

An **element** is a named part of a FHIR resource or datatype.

Examples:

```text
Patient.name
Patient.birthDate
ServiceRequest.subject
CapabilityStatement.fhirVersion
```

An element definition describes its meaning, allowed datatype, cardinality, flags, constraints, and terminology binding.

## Datatype

A **datatype** defines the allowed structure and meaning of an element value.

Primitive examples:

```text
boolean
string
code
date
dateTime
uri
```

Complex examples:

```text
Identifier
HumanName
Coding
CodeableConcept
Reference
Address
```

## Reference

A **Reference** links one FHIR resource to another resource or describes the identity of the intended target.

Literal reference:

```json
{
  "reference": "Patient/123"
}
```

Identifier-based reference:

```json
{
  "identifier": {
    "system": "https://clinic.example/id/patient",
    "value": "PAT-0001"
  }
}
```

`Reference.display` is human-readable text, not authoritative identity.

## Profile

A **profile** is a versioned conformance artifact that constrains or clarifies how a base FHIR resource or datatype is used for a particular context.

A profile may:

- reduce optionality;
- require particular elements;
- restrict allowed datatypes;
- define slicing;
- require terminology bindings;
- add governed extensions;
- add invariants.

A profile does not automatically make a mapping clinically correct.

## Implementation Guide

An **Implementation Guide**, or **IG**, is a published, versioned package of rules and documentation for using FHIR in a specific context.

It may include profiles, extensions, terminology artifacts, examples, operations, search expectations, security guidance, and package dependencies.

An IG must be read with its exact publication version.

## CapabilityStatement

A **CapabilityStatement** is a FHIR conformance resource through which a system advertises how it says it supports FHIR.

It may describe:

- FHIR version;
- supported formats;
- software and implementation information;
- supported resource types;
- interactions;
- search parameters;
- operations;
- security declarations.

Typical metadata request:

```http
GET /metadata
```

A CapabilityStatement is an advertised contract, not complete behavioral proof.

## FHIR server

A **FHIR server** is software that exposes one or more FHIR exchange capabilities.

It may store resources, assign ids, maintain version history, execute searches, run operations, validate resources, and publish a CapabilityStatement.

In this project, HAPI FHIR is the downstream FHIR server.

ClinBridge is not itself a FHIR server implementation.

## Integration engine

An **integration engine** is software focused on receiving, routing, transforming, correlating, acknowledging, monitoring, and reprocessing messages between systems.

It may support MLLP, HL7v2, HTTP, files, XML, JSON, SOAP, and database connectors.

An integration engine is not automatically a FHIR repository or the owner of clinical semantics.

## Resource.id

`Resource.id` is the logical identity of one resource within one FHIR server namespace.

Example:

```text
http://localhost:8080/fhir/Patient/123
```

Here:

```text
resource type = Patient
id            = 123
```

The id is server-local and must not be used as a national identifier, hospital MRN, or source-message id.

## Identifier

An **Identifier** represents a business identity assigned within a known namespace.

Core pair:

```text
system | value
```

Example:

```json
{
  "system": "https://hospital-a.example/id/mrn",
  "value": "12345"
}
```

Equal values in different systems are not automatically equal identities.

Deterministic identifier lookup is not the same as probabilistic patient matching or governed MPI merge.

## Cardinality

**Cardinality** describes how many times an element may or must occur.

```text
0..1  optional, at most once
1..1  required, exactly once
0..*  optional and repeatable
1..*  required at least once and repeatable
```

Cardinality must be read from the element definition, not inferred from one example.

## OperationOutcome

An **OperationOutcome** is a FHIR resource used to report issues, warnings, or errors related to an interaction or operation.

Issues may contain severity, code, diagnostics, expression, and location.

A FHIR client must not assume every failed HTTP request returns a valid OperationOutcome.

## Canonical model

A **canonical model** is an internal normalized representation used to separate source-specific formats from downstream FHIR.

```text
Raw source payload
        !=
Canonical referral
        !=
FHIR resources
```

The canonical model supports deterministic mapping, replay, testing, troubleshooting, and multiple inbound adapters.

## FHIR API

A **FHIR API** applies FHIR interaction rules and representations.

Local base URL:

```text
http://localhost:8080/fhir
```

Examples:

```text
GET  /metadata
POST /Patient
GET  /Patient/{id}
PUT  /Patient/{id}
GET  /Patient?identifier=system|value
```

HAPI's internal SQL schema is not a FHIR API.

## HAPI FHIR

**HAPI FHIR** is a software implementation of FHIR.

In this project, HAPI FHIR JPA Server is used as a local downstream FHIR API and repository.

HAPI owns endpoint behavior, FHIR persistence, resource history, search indexes, its JPA schema, and its CapabilityStatement.

## Liveness

**Liveness** answers whether a process is running and able to respond at a basic level.

Current endpoint:

```http
GET /health
```

```json
{
  "status": "ok"
}
```

This does not prove that PostgreSQL or HAPI is reachable.

## Readiness

**Readiness** answers whether an application can currently perform the work for which it receives traffic.

A process can be live but not ready.

Day 0 keeps `/health` simple and verifies HAPI separately through `/fhir/metadata`.

## Raw payload

A **raw payload** is the original inbound message preserved before normalization or semantic mapping changes it.

Preserving raw input supports evidence, troubleshooting, replay, audit, and mapping-version comparison.

A raw payload must not be repeatedly overwritten with normalized workflow state.

## Mapping

A **mapping** is an explicit transformation from one representation or semantic model to another.

For ClinBridge:

```text
Canonical referral -> FHIR R4 resources
```

A mapping should be deterministic, versioned, testable, traceable to source fields, and honest about ambiguity.

Syntactically valid output does not automatically mean clinically correct output.

## Validation

**Validation** is not one single check.

Planned layers:

```text
Transport and syntax validation
Canonical-model validation
Business-semantic validation
Base FHIR R4 validation
Profile-specific validation later
Terminology validation where justified
```

A resource can pass structural FHIR validation and still represent the wrong clinical workflow.

## Interoperability

**Interoperability** is the ability of independent systems to exchange information and use it with sufficiently consistent structure and meaning.

```text
Successful TCP connection
    does not prove valid HTTP behavior.

Successful HTTP response
    does not prove FHIR support.

Structurally valid FHIR
    does not prove semantic correctness.

Shared code value
    does not prove shared terminology meaning.
```

Interoperability engineering therefore includes transport, contracts, identity, semantics, validation, error behavior, security, and operational ownership.
