# Requester Decision Matrix

## Purpose

This document explains how ClinBridge decides which party should be
used as the requester in a future ServiceRequest.

Day 4 does not create a ServiceRequest. It only records the decision
logic.

## Candidate requester types

| Source meaning | FHIR requester choice | Reason |
| --- | --- | --- |
| Individual clinician requested the service | Practitioner | The source points to the person. |
| Clinician requested the service in an organization role | PractitionerRole | The source carries role-in-organization context. |
| Institution requested the service | Organization | The source points to the institution, not a person. |

## Current Day 4 fixture

The current stored role is:

- PractitionerRole/1058
- practitioner -> Practitioner/1054
- organization -> Organization/1055

This role may be used as requester only when the source says that the
request came from the clinician in that organization role.

## Negative decisions

Do not use PractitionerRole only because both Practitioner and
Organization exist.

Do not use Organization when the source clearly identifies an individual
clinician as the requester.

Do not invent a Practitioner when the source only says that an
institution requested the service.

Do not invent terminology codes for the role without a source system and
a source code.

## Current conclusion

For the Day 4 role graph, PractitionerRole is available as a precise
requester candidate when role-in-organization semantics are present.

The final ServiceRequest.requester value is deferred to Day 5.
