# REST API Security — resource and representation reasoning guide

Use this file when the target exposes REST-like resources and the question depends on how HTTP methods, routes, representations, query controls, or resource relationships map to backend security decisions.

## Goal

Understand the **resource/action model** first, then route each security question to the smallest owning methodology.

REST mechanics are not vulnerabilities by themselves. Method, content-type, pagination, filter, and field-selection differences matter only when they cross a real security boundary.

## Map the resource surface

For the current feature, record:

- resource and identifier/relationship,
- routes and discovered versions representing it,
- methods with real semantic meaning,
- accepted request/response representations,
- actor/auth state,
- parent/tenant relationships,
- query controls such as filter/sort/fields/pagination only where they influence protected data/state,
- authoritative read-back path after mutation.

## Minimal hunt loop

1. Capture one legitimate operation.
2. State the security guarantee under test: authentication, authorization, writable fields, workflow invariant, parser differential, version drift, or injection sink.
3. Change one REST dimension only if it can plausibly reach the same backend capability through a different enforcement path.
4. Verify semantic data/state/action, not status codes or response shape alone.
5. Re-read authoritative state after mutations.
6. Falsify documented alternate representations, public data, no-op methods, schema coercion, cached results, and endpoints that only look equivalent.
7. Hand the confirmed hypothesis to the owning deep-dive rather than continuing generic REST fuzzing.

## High-signal differentials

Use only when the architecture supports them:

- equivalent methods with inconsistent middleware,
- JSON/form/XML/multipart paths handled by different parsers,
- parent/child route forms with different authorization context,
- current vs legacy version of the same operation,
- collection vs individual resource views,
- filter/sort/field selection exposing data beyond the actor's authorized set,
- pagination/limits defeating a concrete data/abuse boundary rather than merely allowing a large response.

## Handoff map

- caller identity / missing auth -> `api_authentication_attacks.md`
- ownership/role/tenant/function boundary -> `api_authorization_bypass.md`
- server-owned fields writable by client -> `api_mass_assignment.md`
- old/current policy drift -> `api_versioning_attacks.md`
- focused parser/method/type variation -> `api_fuzzing_methodology.md`
- rate/velocity accounting -> `api_rate_limiting_bypass.md`
- workflow/business invariant -> `web-mode` business-logic methodology when the key boundary is the product state machine
- exact injection payload -> owning injection skill + `payloads` after a concrete sink exists

## Evidence contract

For a REST-specific differential, preserve:

1. same resource/action mapping,
2. control representation/path/method,
3. changed REST dimension,
4. exact security decision that differs,
5. semantic/durable result,
6. strongest benign explanation tested and rejected.

## Stop conditions

Reject or narrow when:

- only response format/status changes,
- routes/methods represent different legitimate operations,
- broader pagination or field selection still respects the actor's authorization boundary,
- content-type differences only change validation/errors,
- no protected data/action/state becomes available.

This file is a router for REST-specific ambiguity, not a generic exploit checklist.
