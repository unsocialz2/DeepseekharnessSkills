# Account Takeover Chains — evidence-driven impact guide

Use this file **after** a concrete primitive exists and there is evidence it may cross an account-control boundary. Account takeover is an outcome, not a vulnerability class and not a default escalation target.

## Goal

Determine whether the demonstrated primitive can give an attacker durable or meaningful control over another user's account without assuming missing steps.

Do not start by enumerating known ATO recipes. Start from the capabilities the primitive actually grants.

## Capability map

Record the primitive in concrete terms:

- attacker starting privilege,
- victim interaction required,
- origin/session in which the primitive executes,
- data readable,
- actions writable,
- authentication artifacts accessible or unusable,
- whether sensitive actions require password re-entry, MFA, device approval, email confirmation, or another step-up,
- whether sessions are rotated/revoked after account-security changes.

Then ask which account-control guarantees those capabilities can actually challenge.

## Account-control guarantees

Typical guarantees to test only when relevant to the observed product:

- only the legitimate user can change recovery email/phone,
- reset/recovery artifacts are bound to the intended account and flow,
- linking/unlinking identity providers cannot transfer ownership unexpectedly,
- MFA enrollment/removal requires an appropriately strong prior state,
- session/device management cannot be controlled by a weaker session,
- security-sensitive profile changes do not silently alter recovery ownership,
- account merge/link flows preserve verified identity boundaries,
- existing attacker sessions do not survive a security event that should revoke them.

These are reasoning prompts, not a checklist.

## Minimal chain loop

1. Start from a **confirmed primitive** and state exactly what it grants.
2. Identify the nearest account-control boundary that capability could plausibly influence.
3. Verify the prerequisite for that transition from a normal control flow.
4. Test the smallest chain step needed to cross the boundary.
5. After each step, verify server-side account/security state before proceeding.
6. Stop immediately when a required step is independently protected and the primitive cannot satisfy it.
7. If full takeover is demonstrated, reproduce from a clean-enough state and preserve the exact transition sequence.

Never add hypothetical chain steps to make impact look larger.

## Distinguish levels of impact

Keep these separate:

- **action-as-victim** — transient ability to perform same-origin actions in a victim session,
- **security-setting influence** — ability to alter a recovery/security property but not yet obtain durable login,
- **session compromise** — attacker obtains or controls a usable victim session,
- **durable takeover** — attacker can independently regain account access after the original primitive/session ends,
- **tenant/admin takeover** — demonstrated escalation into a stronger organizational role.

Do not label the first two as ATO unless the remaining control boundary is actually crossed.

## High-signal chain questions

Use only when supported by the current capability map:

- Can the primitive alter a recovery factor and does the service require confirmation from the old factor?
- Can it create a new login method or identity-provider link?
- Can it disable/remove MFA or trusted-device protection?
- Can it create/reveal/replay a recovery artifact bound to the victim?
- Do password/recovery/security changes rotate all relevant sessions and tokens?
- Can an account-link/merge flow confuse verified identities across two actors?
- Does a security-sensitive change become durable without step-up authentication?

## Evidence contract

A claimed takeover chain should show:

1. confirmed starting primitive,
2. each account-security transition with request/runtime evidence,
3. actor/account binding at each step,
4. required victim interaction,
5. final proof of attacker-controlled access,
6. whether access survives logout/session expiry/security changes,
7. one negative/control path showing the intended protection.

Use synthetic/test accounts and the minimum state change needed under program rules.

## Stop conditions

Do not continue or claim ATO when:

- the next step requires a secret/factor the primitive cannot obtain or satisfy,
- a security change requires independent re-authentication/step-up that works correctly,
- the primitive provides only self-account control,
- session/action capability disappears before durable control is established,
- account linking correctly binds to a verified external identity,
- evidence supports only a narrower impact.

Report the narrower demonstrated impact instead of forcing an ATO narrative.

## Cross-skill use

The originating flaw stays in its owning skill (XSS, auth, IDOR/BAC, OAuth, MFA, CSRF, business logic, etc.). This file is only for evidence-based impact chaining. Numeric severity and platform wording belong to `reporting-mode`.