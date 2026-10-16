# Authentication Bypass — reasoning guide

Use this file when evidence suggests the application may accept an unauthenticated or insufficiently authenticated actor as a stronger identity. Do not treat this as a generic 403-bypass or header/payload checklist.

## Goal

Identify the application's **authentication state machine** and determine whether any transition to an authenticated/privileged state can occur without satisfying the intended proof of identity.

Authentication bypass is about a broken identity proof or state transition, not merely receiving HTTP 200.

## Model the flow first

For the specific flow under test, map:

- actor state: logged out, partially authenticated, remembered device, password-verified, MFA-pending, fully authenticated, recovery state, SSO callback state, etc.
- credential/proof presented at each step,
- server-side session/token changes after each step,
- redirect/callback parameters and one-time state,
- which component enforces each check: edge/WAF, frontend, gateway, app, identity provider, backend service,
- the protected action/data that demonstrates successful authentication.

The useful question is: **what fact does the server trust as proof that a prior authentication step really happened?**

## Minimal hunt loop

1. Capture one normal control flow from unauthenticated to the strongest state legitimately available.
2. Identify the server-visible state transition at each step.
3. Change one assumption at a time: omit, replay, reorder, substitute, downgrade, cross-account swap, stale-state reuse, or alternate protocol/path only when the observed architecture makes it plausible.
4. Compare the resulting server-side identity and authorization state, not just status codes.
5. Confirm bypass by accessing an action/data set that genuinely requires the stronger identity.
6. Falsify benign explanations such as public fallback content, cached responses, frontend-only redirects, stale browser state, anonymous sessions, or an error body returned with 200.

## High-signal questions

Prefer questions derived from the application over generic tricks:

- Can step N be reached without a server record proving step N-1?
- Is a password/MFA/recovery result bound to the same account/session that consumes it?
- Can a proof generated for actor A be replayed or attached to actor B?
- Can a stale or parallel authentication flow overwrite a newer security state?
- Does a callback trust user-controlled identity/account fields rather than the verified identity from the IdP?
- Does an alternate endpoint/version enforce a weaker authentication precondition for the same protected action?
- Does the edge enforce something that the origin itself does not, creating a real parser/routing discrepancy?

Only investigate method/header/path/protocol normalization when evidence suggests multiple enforcement layers or routing disagreement. Do not spray bypass headers by default.

## Recovery, MFA, SSO and remembered-state boundaries

Treat these as distinct protocols. Record what binds each artifact to:

- account,
- session/browser,
- challenge,
- nonce/state,
- redirect URI/client,
- device,
- expiration/use count.

A weak binding is a candidate only if it can move an attacker into a stronger identity state.

For MFA, distinguish "primary credential accepted" from "fully authenticated". For SSO, distinguish identity-provider success from application account binding. For recovery, distinguish account discovery from actual control.

## Evidence contract

A validated auth bypass should show:

1. initial attacker state,
2. expected authentication prerequisite,
3. exact altered/missing/replayed transition,
4. server-side evidence of the stronger identity/session,
5. protected action/data proving the bypass,
6. a clean control where the prerequisite is enforced normally,
7. the strongest plausible benign explanation tested and rejected.

Do not score severity here; pass demonstrated access/impact facts to `reporting-mode`.

## Stop conditions

Reject or park when:

- only the HTTP status changes while the protected body/action remains unavailable,
- a frontend route is reachable but backend data/actions remain protected,
- behavior depends on the researcher's already-authenticated browser state,
- an alternate method/path returns generic/public content only,
- the observed difference is solely WAF/CDN behavior with no origin-side privilege change,
- the candidate cannot be reproduced from a clean-enough auth state.

## References on demand

Load protocol-specific material only when needed: OAuth/OIDC -> `oauth_misconfig.md`, SAML -> `saml_attacks.md`, MFA -> `2fa_mfa_bypass.md`, JWT -> `jwt_attacks.md`. Use payload/header/path reference material only after the architecture gives a concrete reason to test that differential.