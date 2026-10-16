# GraphQL Security — schema and resolver reasoning guide

Use this file when the target exposes GraphQL and the security question depends on schema discovery, resolver authorization, mutation input authority, batching, or operation-level behavior.

## Goal

Use the schema/operation model to identify **which resolver, object, field, mutation, or relationship enforces a security boundary**.

Introspection or schema visibility is inventory, not a vulnerability by itself.

## Map the GraphQL surface

Build only the schema slice relevant to the current question:

- endpoint and transport/auth state,
- query/mutation/subscription involved,
- object and input types,
- identifiers/relationships used by the resolver,
- fields that expose sensitive state,
- mutation fields that change protected state,
- actor/tenant/role context applied by the backend.

Use introspection when available and permitted. When it is unavailable, prefer schema hints already exposed by client code, docs, errors, or known operations rather than blindly reconstructing the entire schema.

## Minimal hunt loop

1. Capture one legitimate operation and its variables.
2. Identify the exact resolver/security decision under test.
3. Change one actor/resource/field/input relationship at a time.
4. Compare the semantic result and authoritative downstream state.
5. Check whether aliases, batching, fragments, or alternate operations matter only when they exercise the **same security boundary differently**.
6. Falsify public fields, intended graph relationships, client-only hidden fields, null/error masking, and resolver results that do not persist or grant capability.

## High-signal questions

- Is object authorization enforced at every resolver that can reach the object?
- Do nested fields expose data that the parent query's authorization did not intend?
- Can a mutation write fields the actor should not control?
- Are role/tenant checks applied consistently across query and mutation variants?
- Does batching/aliasing defeat a concrete per-operation/account security control?
- Do subscriptions enforce the same identity/tenant boundary over their lifetime?
- Are node/global-ID or alternate lookup resolvers governed by the same policy?

## Introspection

Treat enabled introspection as useful metadata. It becomes security-relevant only if the exposed schema reveals or enables a capability that is otherwise improperly protected; the underlying missing authorization/secret exposure is the finding.

Do not report introspection alone unless the program explicitly treats it as an exposure and the current policy supports that claim.

## Evidence contract

A strong GraphQL finding should preserve:

1. exact operation/field/mutation,
2. actor/auth state,
3. resource/tenant/role relationship,
4. normal control,
5. altered operation/variables,
6. semantic data/state/capability proof,
7. negative/falsification control.

## Stop conditions

Reject or narrow when:

- only schema metadata is exposed,
- the field is intentionally public,
- resolver errors differ without protected data/action becoming available,
- the actor legitimately has the relationship required,
- batching/aliasing changes request shape but not the protected capability.

## Cross-skill use

Use `api_authorization_bypass.md` for BOLA/BFLA, `api_mass_assignment.md` for writable input fields, `api_rate_limiting_bypass.md` only when batching defeats a concrete protected limit, and `payloads` only after a specific downstream parser/injection context is established.
