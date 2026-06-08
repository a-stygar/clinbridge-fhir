# ClinBridge FHIR Glossary

This glossary defines the core terms used in the early ClinBridge slices.

The definitions are intentionally written as practical engineering explanations rather than copied specification text.

## FHIR

**FHIR** stands for **Fast Healthcare Interoperability Resources**.

It is an HL7 interoperability standard that defines reusable healthcare data structures, datatypes, terminology-related structures, references, conformance artifacts, and several exchange patterns.

FHIR is not only:

* a REST API;
* JSON;
* a database schema;
* one vendor product.

REST and JSON are important ways to use FHIR, but the standard also covers XML, operations, messaging, documents, services, profiles, and implementation guidance.

Examples of FHIR resources include `Patient`, `Observation`, `ServiceRequest`, `Bundle`, and `OperationOutcome`.

## Resource

A **resource** is a top-level FHIR exchange unit representing a healthcare, administrative, workflow, infrastructure, or conformance concept.

Every resource instance identifies its type through `resourceType`.

Example:

```json
{
  "resourceType": "Patient"
}
```

Resources have their own identity and lifecycle and can be connected to other resources through references.

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

An element definition describes matters such as:

* meaning;
* allowed datatype;
* cardinality;
* whether it is a modifier;
* constraints;
* terminology binding;
* short and detailed descriptions.

An example JSON document is not sufficient for discovering all element rules. The element definition must be inspected.

## Datatype

A **datatype** defines the allowed structure and meaning of an element value.

Primitive datatype examples:

```text
boolean
string
code
date
dateTime
uri
```

Complex datatype examples:

```text
Identifier
HumanName
Coding
CodeableConcept
Reference
Address
```

A datatype is reusable across many resources. For example, both `Patient` and `Practitioner` can use the `HumanName` datatype.

## Reference

A **Reference** links one FHIR resource to another resource or describes the identity of the intended target.

A literal relative reference may look like:

```json
{
  "reference": "Patient/123"
}
```

An identifier-based reference may look like:

```json
{
  "identifier": {
    "system": "https://clinic.example/id/patient",
    "value": "PAT-0001"
  }
}
```

A reference creates a resource graph without copying the full target resource into every related resource.

`Reference.display` is human-readable text. It is not authoritative identity.

An identifier-based reference may require resolution and must not be assumed to identify exactly one server resource without a defined policy.

## Profile

A **profile** is a versioned conformance artifact that constrains or clarifies how a base FHIR resource or datatype is used for a particular context.

A profile may, within the limits of the base definition:

* reduce optionality;
* require particular elements;
* restrict allowed datatypes;
* define slicing;
* require terminology bindings;
* add governed extensions;
* add invariants.

A profile does not create an unrelated data model under a familiar resource name, and it does not automatically make a mapping clinically correct.

Examples of contexts that may define profiles include a country, healthcare programme, organization, or integration use case.

## Implementation Guide

An **Implementation Guide**, often shortened to **IG**, is a published, versioned package of rules and documentation for using FHIR in a specific context.

An IG may include:

* profiles;
* extensions;
* terminology artifacts;
* examples;
* operations;
* search expectations;
* security guidance;
* narrative implementation instructions;
* package dependencies.

An IG must be read with its exact publication version. A continuous-integration build may differ from a stable published release.

An Implementation Guide is broader than a single profile.

## CapabilityStatement

A **CapabilityStatement** is a FHIR conformance resource through which a system advertises how it says it supports FHIR.

For a REST server it may describe:

* FHIR version;
* supported formats;
* software and implementation information;
* server or client mode;
* supported resource types;
* create, read, update, search, and other interactions;
* search parameters;
* operations;
* security declarations.

The standard REST metadata endpoint is commonly:

```http
GET /metadata
```

A CapabilityStatement is an advertised contract. It does not prove that every interaction has been tested, that every optional behavior is implemented correctly, or that the server is clinically suitable for a particular workflow.

## FHIR server

A **FHIR server** is a software implementation that exposes one or more FHIR exchange capabilities.

A server may:

* expose REST endpoints;
* store FHIR resources;
* assign resource ids;
* maintain version history;
* execute searches;
* run operations;
* validate resources;
* publish a CapabilityStatement.

Not every FHIR server supports every resource, interaction, search parameter, operation, profile, or exchange paradigm.

In the local ClinBridge environment, HAPI FHIR is the downstream FHIR server.

ClinBridge is not itself a FHIR server implementation.

## Integration engine

An **integration engine** is software focused on receiving, routing, transforming, correlating, acknowledging, monitoring, and reprocessing messages between systems.

Depending on the product and configuration, it may support transports and formats such as:

* MLLP;
* HL7v2;
* HTTP;
* files;
* XML;
* JSON;
* SOAP;
* database connectors.

An integration engine is not automatically:

* a FHIR repository;
* the owner of clinical semantics;
* a patient matching authority;
* the correct place for every business rule.

In the planned ClinBridge architecture, Mirth or NextGen Connect will later own MLLP transport, routing, basic HL7v2 checks, minimal normalization, ACK/NACK handling, and message operations. ClinBridge will own the canonical model, semantic FHIR mapping, validation, reliable delivery, and replay policy.

## Resource.id

`Resource.id` is the logical identity of one resource within the namespace of a particular FHIR server.

A resource URL may look like:

```text
http://localhost:8080/fhir/Patient/123
```

In that URL:

```text
resource type = Patient
id            = 123
```

The id:

* is server-local;
* may be assigned by the server;
* participates in literal references such as `Patient/123`;
* may be different on another FHIR server.

It must not be used as a substitute for a national identifier, hospital MRN, source-message id, or other business identifier.

## Identifier

An **Identifier** represents a business identity assigned within a known namespace.

Its core working pair is:

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

The `system` gives the value its namespace and authority context.

These are different identities:

```text
https://hospital-a.example/id/mrn | 12345
https://hospital-b.example/id/mrn | 12345
```

An identifier value without its namespace is often ambiguous.

One Patient may have multiple identifiers from different hospitals, regions, registries, or legacy systems.

Deterministic identifier lookup is not the same as probabilistic patient matching or governed MPI merge.

## Cardinality

**Cardinality** describes how many times an element may or must occur.

Common forms:

```text
0..1
  Optional and may occur at most once.

1..1
  Required and occurs exactly once.

0..*
  Optional and may repeat.

1..*
  Required at least once and may repeat.
```

Cardinality must be read from the element definition, not inferred from one example.

A profile may tighten base cardinality within the limits allowed by the base FHIR definition.

## OperationOutcome

An **OperationOutcome** is a FHIR resource used to report issues, warnings, or errors related to an interaction or operation.

Its issues may include:

* severity;
* issue code;
* diagnostics;
* expression;
* location.

A FHIR client must not assume that every failed HTTP request returns a valid OperationOutcome. A proxy, authentication component, network failure, or broken server may return plain text, HTML, malformed JSON, or no response body.

ClinBridge must preserve transport and HTTP evidence before reducing a failure to a parsed FHIR issue.

## Canonical model

A **canonical model** is an internal normalized representation used to separate source-specific formats from downstream FHIR representation.

For ClinBridge:

```text
Raw source payload
        !=
Canonical referral
        !=
FHIR resources
```

The canonical model is domain-specific. It represents the referral concepts needed by the application without becoming a copy of the source JSON or a copy of FHIR.

This separation supports:

* deterministic mapping;
* source preservation;
* mapping-version changes;
* replay;
* testing;
* troubleshooting;
* multiple inbound adapters.

The canonical model is planned for a later slice and is not implemented during Day 0.

## FHIR API

A **FHIR API** is an interface that applies FHIR interaction rules and representations.

For the local HAPI server, the base URL is:

```text
http://localhost:8080/fhir
```

Examples of FHIR REST interactions include:

```text
GET  /metadata
POST /Patient
GET  /Patient/{id}
PUT  /Patient/{id}
GET  /Patient?identifier=system|value
```

The FHIR HTTP API is the supported boundary between ClinBridge and HAPI.

HAPI's internal SQL schema is not a FHIR API.

## HAPI FHIR

**HAPI FHIR** is a software implementation of the FHIR standard.

In this project, HAPI FHIR JPA Server is used as a local downstream FHIR API and repository.

HAPI owns:

* FHIR endpoint behavior;
* FHIR resource persistence;
* resource version history;
* search indexes;
* its JPA persistence schema;
* its CapabilityStatement.

Using HAPI does not make ClinBridge a HAPI plugin, and ClinBridge must not depend on HAPI internal database tables.

## Liveness

**Liveness** answers whether a process is running and able to respond at a basic level.

The current ClinBridge endpoint:

```http
GET /health
```

returns:

```json
{
  "status": "ok"
}
```

This proves only basic process-level HTTP availability.

It does not prove that PostgreSQL or HAPI is reachable.

## Readiness

**Readiness** answers whether an application can currently perform the work for which it receives traffic.

A future ClinBridge readiness check may consider:

* required configuration;
* ClinBridge database connectivity;
* HAPI availability;
* required downstream capability.

A process can be live but not ready.

Day 0 keeps `/health` simple and verifies HAPI separately through `/fhir/metadata`.

## Raw payload

A **raw payload** is the original inbound message preserved before normalization or semantic mapping changes it.

Preserving raw input supports:

* evidence;
* troubleshooting;
* replay;
* mapping-version comparison;
* audit;
* investigation of source-contract changes.

A raw payload must not be repeatedly overwritten with normalized workflow state.

Only synthetic payloads are permitted in the current public repository.

## Mapping

A **mapping** is an explicit transformation from one representation or semantic model to another.

In ClinBridge, the main planned mapping is:

```text
Canonical referral -> FHIR R4 resources
```

A mapping should be:

* deterministic;
* versioned;
* testable;
* traceable to source fields;
* honest about missing or ambiguous semantics.

Syntactically valid output does not automatically mean that the mapping is clinically correct.

## Validation

**Validation** is not one single check.

ClinBridge plans several separate validation layers:

```text
Transport and syntax validation
Canonical-model validation
Business-semantic validation
Base FHIR R4 validation
Profile-specific validation later
Terminology validation where justified
```

A resource can pass structural FHIR validation and still represent the wrong clinical workflow.

Deterministic validation errors should not be retried without changing the data or configuration.

## Interoperability

**Interoperability** is the ability of independent systems to exchange information and use it with sufficiently consistent structure and meaning.

Technical connectivity alone is not full interoperability.

For example:

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
