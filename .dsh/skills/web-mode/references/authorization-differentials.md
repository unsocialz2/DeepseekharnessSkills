# Authorization Differentials — on-demand inspiration

Load only after a real actor/resource/action boundary is identified. This preserves useful differential ideas without turning IDOR/BAC testing into identifier spraying.

## Alternate representations

When the same resource or action exists through more than one representation, compare the authorization decision across relevant variants such as:
- route aliases or API versions,
- parent/child nesting,
- collection vs single-object endpoints,
- export/share/download views,
- client-visible and internal identifiers,
- alternate serializers/content types only when they reach the same backend action.

## Identifier forms

Numeric IDs, UUIDs, slugs, hashes, composite keys, and capability URLs are implementation details. Change identifier form only when the application already exposes multiple forms or when discoverability/capability semantics are part of the boundary.

## Action asymmetry

Authorization may differ by action. Compare read, update, delete, share, export, approve, or role-changing actions only when they apply to the same resource relationship.

## Parent / tenant binding

Check whether child resources remain bound to the parent tenant/owner when route or body references disagree. The security question is which server-side relationship is authoritative.

## Capability lifecycle

Share links, signed URLs, invitations, export links, and other capabilities should be scoped and revocable according to product intent. Pay attention when ownership, membership, revocation, expiration, or object state changes.

## Use rule

Pick one differential justified by observed architecture, keep a clean control, and verify semantic success independently. A different status code or identifier shape is not an authorization finding by itself.
