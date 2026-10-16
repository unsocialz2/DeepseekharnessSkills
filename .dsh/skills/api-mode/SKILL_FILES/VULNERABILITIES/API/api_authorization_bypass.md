# API Authorization — actor/resource/action reasoning guide

Use this file when the API authenticates the caller but may enforce ownership, tenant, role, relationship, or function-level permissions incorrectly.

## Goal

Determine whether the server enforces **which authenticated actor may perform which action on which resource**.

Do not reduce BOLA/BFLA testing to changing identifiers or comparing status codes.

## Build the minimum matrix

For the current question, record:

```text
ACTOR A: <role/account/tenant>
ACTOR B: <other role/account/tenant>
RESOURCE A: <owned/accessible by A>
RESOURCE B: <owned/accessible by B>
ACTION: <read/update/delete/export/share/approve/...>
```

Establish normal controls, then change one actor/resource/action relationship at a time.

## Reason from the authorization source

Ask:

- What server-side fact should authorize this action: ownership, membership, tenant, role, capability, relationship, workflow state?
- Where is that fact derived: session/token claims, database relation, route parent, body field, cached relation, service-to-service context?
- Is the decision applied consistently across CRUD actions and alternate representations?
- Do collection/list/export/share endpoints expose or grant a stronger capability than object endpoints?
- Can parent/child identifiers disagree and escape tenant/ownership binding?
- Do old/alternate API versions or GraphQL/gRPC operations reach the same resource through a different policy path?

## Minimal hunt loop

1. Choose one resource/action boundary.
2. Prove allowed behavior for the rightful actor.
3. Change exactly one actor/resource/role relationship.
4. Verify semantic result: protected data returned, state changed, action executed, or capability granted.
5. Re-read authoritative state independently when possible.
6. Falsify public/shared resources, intended collaboration, stale cache, aliases, soft-deleted visibility, or misleading 2xx/error bodies.
7. Expand only when a real boundary failure or shared policy path is established.

## Horizontal, vertical, contextual

Keep distinct:

- **horizontal**: wrong peer owner/tenant,
- **vertical**: weaker role invokes stronger function,
- **contextual**: action allowed only in a particular relationship/workflow state.

The identifier format is secondary; discoverability is not the same as authorization.

## Evidence contract

A strong finding should preserve:

1. actor/resource/action matrix,
2. normal control,
3. cross-boundary request/result,
4. semantic proof of unauthorized capability,
5. ownership/tenant/role evidence,
6. independent state verification where relevant,
7. one falsification/negative control.

Numeric severity belongs to `reporting-mode`.

## Stop conditions

Reject or narrow when:

- the resource/action is intentionally shared/public to that actor,
- only identifier predictability is shown,
- status changes without protected semantic success,
- the actor already has a legitimate relationship granting access,
- an apparent mutation produces no authoritative state change.

## Cross-skill use

Use `api_mass_assignment.md` when the issue is writable fields rather than actor/resource permission, `api_versioning_attacks.md` when policy drift is version-specific, and `business_logic_bugs.md` when workflow invariants—not simple ownership/role—define the boundary.
