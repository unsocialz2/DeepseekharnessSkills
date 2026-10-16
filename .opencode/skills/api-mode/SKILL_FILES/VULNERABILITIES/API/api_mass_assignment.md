# API Mass Assignment — writable-field authority reasoning guide

Use this file when an API accepts object-like input and may bind client-supplied fields directly to server-side models or state.

## Goal

Determine whether the client can write a field whose value should be **derived, protected, role-gated, server-owned, or immutable**.

Do not equate “undocumented field accepted” with a vulnerability unless it changes protected state or authority.

## Model the object boundary

For the relevant resource, identify:

- fields the client is expected to control,
- fields returned but intended read-only,
- server-derived identity/tenant/owner fields,
- role/security/status/financial fields,
- create-only vs update-only vs immutable fields,
- schema/OpenAPI/GraphQL hints where available,
- authoritative post-write state.

## Minimal hunt loop

1. Capture a normal create/update request and response.
2. Compare visible response/schema fields with writable input fields.
3. Select one field whose authority is security-relevant and plausibly mapped by the backend.
4. Add/change that field while keeping the request otherwise valid.
5. Re-read the resource independently to confirm authoritative state.
6. Falsify ignored/echo-only fields, client-side display artifacts, asynchronous rollback, or a field the current actor is legitimately allowed to set.

## High-signal questions

- Does the server derive owner/tenant/role/status, or accept a client interpretation?
- Are read-only fields actually write-protected?
- Do create, PUT, PATCH, and bulk-update paths expose different writable sets?
- Does nested input update related objects or security-sensitive subfields unexpectedly?
- Do GraphQL input types or alternate API versions expose fields hidden by the normal client?

## Evidence contract

A strong finding should preserve:

1. actor/resource context,
2. protected field and intended authority,
3. normal control request,
4. altered request,
5. authoritative persisted state,
6. resulting capability/impact,
7. negative/falsification control.

Numeric severity belongs to `reporting-mode`.

## Stop conditions

Reject or narrow when:

- the field is ignored or only echoed,
- the actor is intended to control the field,
- the change does not persist or affect authoritative state,
- only cosmetic/client-local behavior changes.

## Cross-skill use

Use `api_authorization_bypass.md` when the primary issue is which actor may mutate the object, and `business_logic_bugs.md` when the field change violates a broader workflow/business invariant.
