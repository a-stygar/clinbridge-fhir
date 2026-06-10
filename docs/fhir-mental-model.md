# FHIR Mental Model — Day 1

## Resource and element

A FHIR resource is a top-level exchange object that represents a healthcare or administrative concept, such as `Patient`, `Practitioner`, or `ServiceRequest`.

Every resource contains a `resourceType` and consists of named elements. Each element has a defined meaning, datatype, cardinality, and possibly additional constraints.

For example, `Patient.birthDate` and `Patient.identifier` are elements of the `Patient` resource.

## Primitive and complex datatype

A primitive datatype is usually represented by a single JSON scalar value.

Examples include:

* `boolean`
* `string`
* `date`
* `dateTime`
* `uri`
* `code`

Primitive values still have datatype-specific rules. For example, a FHIR `date` must use the required lexical format, such as `1975-03-12`.

A complex datatype contains multiple related elements.

Examples include:

* `Identifier`
* `HumanName`
* `Address`
* `CodeableConcept`
* `Reference`

For example, an `Identifier` may contain `system`, `value`, `use`, and other elements.

`Patient.gender` is an element whose datatype is `code`. Its allowed values are constrained by the element definition and its value-set binding.

## Cardinality

Cardinality defines the minimum and maximum number of occurrences allowed for an element.

Common examples are:

```text
0..1 = optional, at most one occurrence
1..1 = required, exactly one occurrence
0..* = optional and repeatable
1..* = at least one occurrence and repeatable
```

In FHIR JSON, repeatable elements are represented as arrays, even when only one item is present.

For example, `Patient.identifier` and `Patient.name` are repeatable and therefore use JSON arrays.

## Resource.id vs Identifier

`Resource.id` is the logical identity of one resource within a particular FHIR server and base URL.

For example:

```text
Patient/1001
```

The same real-world person may be represented by another resource id on another FHIR server.

An `Identifier` represents a business identity assigned by an issuing authority or source system.

For example:

```text
system = https://clinic.example.it/id/patient
value  = PAT-0001
```

One Patient can have multiple identifiers because the same person may have identifiers from different hospitals, laboratories, registries, or national systems.

## Why identifiers need namespaces

The `system` element identifies the namespace or issuing identifier system in which the `value` has meaning.

The same value may be issued independently by different organizations:

```text
Hospital A MRN | 12345
Hospital B MRN | 12345
```

These are not automatically the same identifier.

An identifier without `system` may still be structurally accepted in some FHIR contexts, but it is ambiguous and is not sufficient for reliable deterministic matching across systems.

For interoperability, the meaningful working identity is normally the pair:

```text
system | value
```

## What HAPI added on create

The submitted Patient did not contain a server id or metadata.

After the create interaction, HAPI added:

```text
id = 1001
meta.versionId = 1
meta.lastUpdated = 2026-06-10T13:50:19.693+00:00
```

The original business elements were returned with the same values:

```text
identifier
name
gender
birthDate
```

The JSON property order is not part of the semantic contract.

The create response also included:

```text
Location
ETag
Last-Modified
```

`Location` identified the REST location of the created resource version. `ETag` represented the current version identifier, and `Last-Modified` recorded when that resource version was last changed.

## Observed invalid-date behavior

The following payload was valid JSON but contained an invalid FHIR `date` lexical value:

```json
{
  "resourceType": "Patient",
  "birthDate": "12/03/1975"
}
```

Observed response:

```text
HTTP status: 400
Content-Type: application/fhir+json;charset=UTF-8
Response resourceType: OperationOutcome
issue.severity: error
issue.code: processing
```

The diagnostics reported that `12/03/1975` was not a valid FHIR date/time format.

This was a deterministic payload error, not a transport failure or transient server failure. Retrying the same unchanged request would not fix it.

This result demonstrates the observed behavior of the current HAPI configuration for this invalid value. It does not prove that an ordinary create interaction performs every possible profile, terminology, or business validation check.
