# Resource Graph: Parties

## Stored resources

- Practitioner/1054 represents the individual clinician.
- Organization/1055 represents the healthcare organization.
- PractitionerRole/1058 represents the clinician's role in that organization.

## Graph edges

PractitionerRole/1058 has these references:

- practitioner -> Practitioner/1054
- organization -> Organization/1055

This means the role is not the same thing as the person.
It is also not the same thing as the organization.
It connects the person to the organization in a specific role context.

## Evidence

The PractitionerRole was read back from HAPI and contained:

- practitioner.reference = Practitioner/1054
- organization.reference = Organization/1055

A search using both references returned one match:

- PractitionerRole/1058

## Semantic note

Practitioner identifies the individual person.
Organization identifies the institution.
PractitionerRole identifies the person's role in that institution.

The role code is represented as text only.
No external terminology system is claimed.
