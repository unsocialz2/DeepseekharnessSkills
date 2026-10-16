# Business Logic Bugs — invariant-driven reasoning guide

Use this file when the application behaves correctly at the HTTP/security-control level but may allow a sequence or state that violates the product's intended business rules.

This skill should **increase creativity**, not reduce the hunt to a list of familiar bug patterns.

## Goal

Discover a state that the product considers impossible, unauthorized, economically invalid, or inconsistent — then prove an attacker can reach it through allowed-looking operations.

Do not begin from payloads. Begin from **actors, assets, state, transitions, and invariants**.

## Build the smallest useful model

For the feature under test, identify:

- **actors** — customer, guest, owner, collaborator, merchant, support/admin, external service, background worker, etc.
- **assets** — account, booking, order, balance, entitlement, collection, coupon, subscription, workflow object, approval, refund, quota, reputation, inventory.
- **states** — only the states relevant to the current flow.
- **transitions** — which action moves the asset from one state to another.
- **authority** — who is supposed to cause each transition.
- **invariants** — statements that should always remain true.

Examples of invariants are useful only as form, not as a checklist:

- an actor can modify only objects they are entitled to control,
- value cannot be created from nothing,
- a completed/revoked object cannot regain capabilities unexpectedly,
- one-time effects happen once,
- a later workflow step implies earlier prerequisites were satisfied,
- changing identity/ownership does not preserve privileges that belonged to the previous identity,
- two individually valid operations should not combine into an invalid state.

Write the target-specific invariant in your own words before testing it.

## Hunt from contradictions

Ask questions that challenge the product's assumptions:

1. What does component B assume component A already validated?
2. Which state is represented differently across frontend, API, worker, cache, payment provider, or another service?
3. Can two valid operations be ordered, repeated, interleaved, or performed by different actors to break an invariant?
4. What remains usable after cancel, revoke, transfer, downgrade, merge, logout, expiration, or completion?
5. Which identifier or state is trusted because the UI normally hides or sequences it?
6. Can a capability survive the lifecycle event that should have destroyed it?
7. Does the server derive ownership/price/role/state, or accept a client-supplied interpretation of it?
8. Are there parallel representations of the same business fact that can drift apart?

Let observed architecture choose the next question. Do not mechanically run negative-number, race, replay, coupon, or step-skip tests unless the flow makes them plausible.

## Minimal experiment design

For each hypothesis:

1. State the invariant you think may be broken.
2. Establish a normal control demonstrating the intended rule.
3. Change the minimum number of variables needed to challenge that rule.
4. Observe the **durable business state**, not only the immediate response.
5. Verify downstream consequences from an independent read/view when possible.
6. Try the strongest benign explanation: eventual consistency, idempotent response, UI-only stale state, sandbox behavior, delayed rollback, intended sharing/delegation, or documented product behavior.
7. Stop if the state self-corrects or no meaningful attacker advantage remains.

## Cross-actor and lifecycle reasoning

Many high-value logic bugs appear at boundaries rather than within one request. Pay special attention when evidence shows:

- ownership changes,
- guest -> registered transitions,
- pre-action -> completed object transitions,
- invited/collaborator -> owner distinctions,
- trial -> paid -> downgraded states,
- pending -> approved/rejected transitions,
- external provider callbacks,
- asynchronous jobs,
- object cloning/import/export,
- account merge/link/unlink,
- cancellation/refund/rebooking flows.

Do not assume these are vulnerable; use them as prompts to inspect where invariants cross components.

## Impact reasoning

Describe impact as the broken business guarantee plus demonstrated attacker advantage:

- unauthorized control/access,
- durable entitlement gain,
- loss/creation/transfer of value,
- bypassed approval or policy state,
- cross-tenant/customer effect,
- irreversible or repeatable workflow corruption.

Avoid speculative financial numbers or severity labels during hunting. Quantify only what the evidence actually demonstrates.

## Evidence contract

A strong business-logic finding should preserve:

1. target-specific invariant,
2. normal/control flow,
3. minimal violating sequence,
4. actor/object identities relevant to the boundary,
5. durable post-condition proving the invalid state,
6. strongest benign explanation tested and rejected,
7. real attacker advantage/impact.

## Stop conditions

Park or reject when:

- behavior is documented/clearly intended,
- only the UI is inconsistent while authoritative state remains correct,
- a temporary state is rolled back before any attacker benefit,
- the hypothesis requires an unavailable privileged actor or impossible prerequisite,
- the effect is cosmetic or self-only with no meaningful security/business consequence,
- repeated testing produces no new evidence after the current time-box.

## Inspiration on demand

If the state model stalls, consult `references/business-logic-patterns.md` for historical invariant families as **inspiration only**. Choose at most one family that matches observed architecture, rewrite it as a target-specific invariant, and return to the evidence-driven experiment loop. Never treat the examples as the test plan.
