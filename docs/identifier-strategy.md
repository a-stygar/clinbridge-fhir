# Identifier Strategy

## Purpose

This document defines how ClinBridge treats FHIR business identifiers in the Day 3 synthetic lab.

ClinBridge treats a business identifier as a pair:

```text
Identifier.system + Identifier.value
```

`Identifier.value` alone is not a safe identity key because the same visible value may be issued by different systems.

All identifiers in this lab are synthetic. The example URI systems are portfolio-owned example namespaces and do not claim to represent official Italian national identifier conventions.

## Identifier Catalogue

| Business concept | Example system URI | Example value | Issuer/owner | Expected uniqueness scope | How ClinBridge may use it | What ClinBridge must not infer |
|---|---|---|---|---|---|---|
| Patient source identifier | `https://clinic.example.it/id/codice-fiscale` | `SYNTHETIC-CF-0001` | Synthetic clinic source system | Unique only within this synthetic clinic identifier namespace | Use as a governed patient lookup key when the source contract says this namespace is stable | Do not infer that this is a real codice fiscale, globally valid national identifier, or proof of a real person |
| Patient hospital MRN | `https://hospital.example.it/id/mrn` | `MRN-0001` | Synthetic hospital | Unique only within that hospital MRN namespace | Use with token search as `system|value` to find a Patient candidate from that hospital namespace | Do not treat the same MRN value from another hospital as the same identifier |
| Patient MRN from another hospital | `https://other-hospital.example.it/id/mrn` | `MRN-0001` | Second synthetic hospital | Unique only within the other hospital MRN namespace | Demonstrate namespace separation for equal-looking values | Do not merge or link patients just because the visible value is also `MRN-0001` |
| Practitioner professional identifier | `https://clinic.example.it/id/practitioner-license` | `PRACT-001` | Synthetic clinic credentialing source | Unique only within the practitioner-license namespace | Use as a deterministic Practitioner lookup key when governed by the source contract | Do not infer employment, role, specialty, or organization context from the identifier alone |
| Organization identifier | `https://clinic.example.it/id/organization` | `HOSP-0001` | Synthetic organization registry | Unique only within the organization identifier namespace | Use as a deterministic Organization lookup key | Do not use organization name alone as stable identity and do not infer legal identity beyond this synthetic namespace |

## Lookup Policy

### Zero Matches

Zero matches means no known resource was found for the searched identifier pair.

ClinBridge may create, reject, or escalate the record depending on the later workflow contract. Zero matches is a valid search outcome, not a transport or FHIR parsing failure.

### One Match

One match means one candidate resource was found for the searched identifier pair.

ClinBridge may treat this as a deterministic candidate only when the namespace is governed and after verifying:

```text
resourceType
expected Identifier.system
expected Identifier.value
```

One exact identifier match does not by itself prove that two records should be merged.

### Multiple Matches

Multiple matches means the identifier pair is ambiguous in the current server state.

ClinBridge must not select the first entry automatically. Automatic linking must stop, and the case must be escalated or reconciled according to a governed workflow.

## Token Search Rule

When the namespace is known, ClinBridge should search identifiers using token search with both system and value:

```text
identifier=system|value
```

Value-only search can be broader and unsafe for automatic identity decisions because the same value may exist in multiple identifier systems.

## Explicit Limitations

ClinBridge can perform deterministic lookup using governed identifiers.

ClinBridge does not claim probabilistic patient matching, MPI ownership, demographic matching, merge authority, or clinical identity governance.

A base FHIR server may contain duplicate business identifiers unless stronger profile, server configuration, or workflow rules prevent them.

Identifier normalization rules, such as stripping punctuation, changing case, or removing leading zeros, must come from a source-owner contract rather than developer intuition.

Organization names are human-readable labels, not stable identifiers.
