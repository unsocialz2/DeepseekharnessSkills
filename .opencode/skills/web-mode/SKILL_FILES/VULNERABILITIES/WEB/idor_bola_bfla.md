# IDOR / BOLA / BFLA — authorization reasoning guide

Use this file for browser/web flows where a user-controlled reference, action, or workflow may cross an ownership/role boundary. API-wide authorization testing belongs to `api-mode`.

## Goal

Determine whether the server enforces **who may perform which action on which resource**, independent of what identifiers or UI controls the client exposes.

Do not reduce authorization testing to changing an ID and comparing `200` vs `403`.

## Build the minimum actor-resource-action matrix

For the current feature, record only what you need:

```text
ACTOR A: <role/account>
ACTOR B: <other role/account>
RESOURCE A: <owned by A>
RESOURCE B: <owned by B>
ACTION: <read/update/delete/share/export/...>
```

Establish normal controls first:

- A -> resource A: expected allowed behavior,
- B -> resource B: expected allowed behavior,
- A -> resource B: the actual boundary test,
- optional role/privilege control only when vertical authorization is relevant.

The identifier format (numeric, UUID, slug, hash, composite key) is secondary. Knowing another object's identifier is not itself a vulnerability.

## Reason from the authorization decision

Ask:

- What server-side fact should authorize this action: ownership, membership, tenant, role, relationship, capability, share grant, workflow state?
- Where does that fact come from: session, token claims, database relation, object field, route parent, client-supplied body, cached state?
- Is authorization checked consistently for read vs mutation vs delete vs export/share actions?
- Can the same object be reached through another route/version/representation with a different enforcement path?
- Does changing parent/child identifiers break tenant or ownership binding?
- Do collection/list endpoints expose capabilities or object references that bypass the intended discovery boundary?
- Does a state transition change ownership/visibility while stale capabilities remain usable?

Test the question the architecture suggests; do not spray alternate encodings, headers, methods, or body formats without evidence of multiple parsers/enforcement layers.

## Minimal hunt loop

1. Choose one concrete action and resource boundary.
2. Prove normal ownership/role behavior with a clean control.
3. Change exactly one actor/resource relationship.
4. Verify the **semantic result**: sensitive data returned, object changed, action executed, or capability granted.
5. Re-read the resource/state independently when possible to prove durable effect.
6. Falsify alternate explanations such as public/shared data, stale cache, object aliases, intended collaboration, soft-deleted visibility, or error pages with misleading status codes.
7. Expand to sibling actions only if the first boundary failure is real or the architecture clearly shares the same authorization path.

## Horizontal vs vertical boundaries

Keep them distinct:

- **horizontal** — same privilege level, wrong owner/tenant/object relationship,
- **vertical** — lower-privileged actor can invoke an action/function reserved for a stronger role,
- **contextual** — action is allowed only in a particular relationship or workflow state.

A read failure does not imply a write failure and vice versa. Test the smallest meaningful action supported by evidence.

## Identifier discovery vs authorization

Do not spend time making identifiers predictable unless discoverability is actually part of the security boundary.

If the application legitimately exposes an identifier but still denies unauthorized access, authorization is working. If a list/search/share endpoint grants a capability or cross-user object reference, analyze that capability directly rather than calling predictability the bug.

## Evidence contract

A strong authorization finding should preserve:

1. actor/resource/action matrix,
2. expected control request/result,
3. cross-boundary test request/result,
4. semantic proof of unauthorized data/action,
5. ownership/tenant/role evidence for both sides,
6. one falsification/negative control,
7. scope and victim/reach facts.

Do not assign numeric severity here. Pass demonstrated data/action/role impact to `reporting-mode`.

## Stop conditions

Reject or narrow the claim when:

- the resource is intentionally public/shared,
- the returned data is intentionally available to the same actor through another supported surface and no stronger capability is gained,
- a status code changes but no protected data/action becomes available,
- the test account legitimately has a relationship granting access,
- an action appears successful but independent state verification shows no effect,
- the only issue is identifier predictability with correct authorization,
- the candidate depends on an unavailable role or fabricated ownership assumption.

## Cross-skill use

Use `api-mode` for broad REST/GraphQL/gRPC authorization matrices, `business_logic_bugs.md` when the boundary depends on workflow/invariants rather than simple ownership/role, and `account_takeover_chains.md` only after a real authorization primitive is demonstrated.

If architecture suggests alternate representations, parent/tenant binding issues, capability-lifecycle differences, or action asymmetry, consult `references/authorization-differentials.md` as **inspiration only** rather than a spray list.
