# Business Logic Pattern Families — inspiration only

Load this file only when the target-specific state model stalls. These are examples of invariant shapes, not a checklist.

## Value consistency

Common invariant shapes:
- value should not be created or duplicated unexpectedly,
- quantities, prices, credits, refunds, and rewards should stay within valid bounds,
- balances and entitlements should agree across services,
- one logical transaction should have one authoritative financial outcome.

## One-time effects

Ask whether an effect intended to occur once remains once-only across retries, duplicate callbacks, alternate routes, parallel processing, and partial failures.

## Workflow ordering

Look for contradictions such as:
- a later state existing without its prerequisite,
- completed, cancelled, expired, or revoked objects regaining capability unexpectedly,
- approval/check applied to one state while a later mutation uses another,
- stale capabilities surviving a lifecycle change.

## Identity and ownership transitions

High-signal transitions include guest-to-account, invitee-to-owner, transfer, merge/link/unlink, tenant movement, recovery-detail changes, and object cloning/import/export.

The useful question is whether capabilities issued under the old identity or ownership remain valid after authority changes.

## Subscription and entitlement state

Check whether billing state and product capability remain consistent across trial, paid, downgrade, cancellation, seat changes, add-ons, quotas, and delayed payment outcomes.

## Asynchronous state

Cross-component assumptions often fail around background jobs, callbacks, queues, retries, cache invalidation, review decisions, inventory reservations, and delayed verification.

Test the durable final state rather than the immediate response.

## Concurrency

Concurrency matters when an invariant depends on a check occurring before a mutation. Establish a sequential control first; only then ask whether synchronized otherwise-valid operations can create a state that is impossible sequentially.

## Discovery vs capability

Separate harmless identifier discovery from actual capability. List/search/share surfaces matter when they reveal or preserve a capability that should be scoped, revocable, or actor-bound.

## Use rule

Choose at most one pattern family that fits observed architecture. Rewrite it as a target-specific invariant, then design the smallest experiment that could disprove that invariant. If none fits, return to the product's own state model rather than forcing an example.
