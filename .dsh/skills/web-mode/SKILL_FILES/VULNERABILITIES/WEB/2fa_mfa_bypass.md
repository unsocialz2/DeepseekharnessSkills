# 2FA / MFA Bypass — state and binding reasoning guide

Use this file when the product has a distinct multi-factor or step-up authentication state and evidence suggests that state may not be enforced consistently.

MFA bypass is not "find any endpoint that works before OTP." The question is whether the server accepts an action that should require a stronger authentication state without the intended second factor.

## Goal

Model how the application represents MFA state, what the second factor is bound to, and which actions require that state.

A valid attacker model can be a party who already has the primary credential/session but **does not possess the required second factor**. Testing with a controlled account is sufficient to demonstrate that boundary; a second test account is not inherently required.

## Map the MFA state machine

For the relevant flow, record:

- primary-authenticated but MFA-pending state,
- fully authenticated state,
- remembered/trusted device state if present,
- step-up state for especially sensitive actions,
- recovery/fallback state,
- session/token/cookie changes between these states,
- what account/challenge/session/device each verification artifact is bound to.

The key question is: **what server-side evidence proves that this exact session/account completed the required factor?**

## Minimal hunt loop

1. Capture one normal MFA flow and note all server-visible state changes.
2. Identify one protected action that demonstrably requires the stronger state.
3. Change one assumption at a time: skip, replay, reorder, cross-session swap, stale-state reuse, alternate login path, or recovery/fallback path only when the architecture makes it plausible.
4. Verify whether the server-side authentication state actually changes or whether only the UI changes.
5. Confirm the result by performing the protected action from the tested state.
6. Falsify benign explanations such as remembered-device policy, grace periods, already-elevated sessions, account-specific MFA exemptions, or cached frontend state.

## Binding questions

High-signal questions include:

- Is the factor result bound to the same account that consumes it?
- Is it bound to the same session/browser/challenge?
- Can a proof from one parallel login flow affect another?
- Does verification upgrade only one token/session, or are weaker sessions accidentally treated as upgraded too?
- Are alternate login methods subject to the same MFA policy?
- Does disabling/changing MFA require a state at least as strong as enabling it?
- Are recovery/fallback flows equivalent in assurance to the normal MFA path?
- Are step-up requirements checked server-side on the sensitive action itself?

Do not brute-force or flood verification endpoints unless program rules explicitly allow it and the security question actually depends on rate limiting. Prefer low-volume evidence about binding and state enforcement first.

## Recovery and fallback

Treat backup codes, support recovery, trusted devices, alternate identity providers, and phone/email fallback as separate authentication protocols. Analyze their guarantees directly instead of assuming they are weaker.

A fallback path is only a bypass if it grants the same protected state without satisfying the product's intended assurance level.

## Evidence contract

A validated MFA bypass should show:

1. starting auth state,
2. factor normally required,
3. exact altered/skipped/replayed transition,
4. resulting server-side auth state,
5. protected action proving the stronger state was accepted,
6. normal control path where the factor is enforced,
7. account/session/challenge binding evidence,
8. strongest benign explanation tested and rejected.

Do not jump directly to ATO. If the bypass only grants a narrower action, record that narrower impact. Use `account_takeover_chains.md` only when an evidence-backed account-control path remains.

## Stop conditions

Reject or narrow when:

- only client-side navigation changes,
- the sensitive backend action still enforces MFA/step-up,
- the tested session was already trusted/elevated before the test,
- alternate login is intentionally exempt by documented policy and does not weaken another boundary,
- a stale code/session response does not result in stronger server-side capability,
- the tested state gains no capability beyond what the intended primary-authenticated/MFA-pending state is supposed to have.

## Cross-skill use

Use `auth_bypass_all_types.md` for the wider login state machine, `rate_limiting_bypass.md` only when rate limiting is the actual boundary under test, OAuth/SAML/JWT files for protocol-specific questions, and `account_takeover_chains.md` only after a real primitive is confirmed.
