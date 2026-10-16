# API Versioning — security-drift reasoning guide

Use this file when multiple discovered API versions or compatibility routes implement the same resource/action and may enforce different security guarantees.

## Goal

Determine whether a **reachable alternate version** exposes the same protected capability with weaker authentication, authorization, validation, lifecycle, or business-state enforcement.

Do not brute-force arbitrary version strings as the primary methodology.

## Build a version map from evidence

Prefer versions discovered from:

- current/old clients or mobile apps,
- OpenAPI/Swagger/docs,
- JavaScript bundles,
- traffic/history/archive artifacts,
- response links/errors/headers,
- target-local recon.

For each relevant version, map only the operation needed for the current question.

## Minimal hunt loop

1. Pick one security-sensitive operation that exists in the current version.
2. Establish its normal security behavior.
3. Identify a discovered alternate/legacy version implementing the same or equivalent operation.
4. Replay the minimum semantically equivalent test with the same actor/resource state.
5. Compare the security decision and durable result, not only schema/status differences.
6. Falsify intentionally different product semantics, deprecated read-only endpoints, test/sandbox environments, and unrelated resources sharing similar names.

## High-signal drift

Relevant differences may include:

- missing/weaker authentication,
- inconsistent object/function authorization,
- writable fields protected in the newer version,
- missing step-up or state prerequisites,
- weaker parser/validation path for the same business action,
- stale/revoked capabilities remaining valid only on the old path.

Rate limits or pagination differences are security-relevant only when they protect a concrete invariant or data boundary.

## Evidence contract

A strong finding should preserve:

1. discovered-version evidence,
2. equivalent operation/resource mapping,
3. current-version control,
4. alternate-version test,
5. exact security guarantee that differs,
6. semantic/durable impact,
7. strongest benign explanation tested and rejected.

## Stop conditions

Reject or narrow when:

- the alternate version was guessed but not meaningfully exposed/reachable,
- versions implement different products/resources rather than the same boundary,
- only formatting/schema/pagination differs,
- the legacy path is correctly disabled or preserves the same security decision,
- scope/rules exclude the alternate asset/version.

## Cross-skill use

Once version drift identifies the broken boundary, use the owning skill—authentication, authorization, mass assignment, business logic, parser/injection, or rate limiting—to validate that boundary. This file identifies **where policy drift exists**, not a separate exploit class.
