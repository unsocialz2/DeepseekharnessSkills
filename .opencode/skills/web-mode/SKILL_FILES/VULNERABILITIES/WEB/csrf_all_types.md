# CSRF — browser authority and request-binding reasoning guide

Use this file when a browser automatically supplies credentials to a state-changing endpoint and evidence suggests the server may accept a cross-site request without the intended request-origin/session binding.

## Goal

Determine whether an attacker-controlled site/context can cause the victim's browser to perform a **security-relevant state change using ambient victim authority** without the intended anti-CSRF protection.

Do not equate “request succeeds without a token” with CSRF until the cross-site browser delivery conditions are also proven.

## Model the browser/request boundary

For the action under test, identify:

- credential type automatically attached by the browser,
- cookie SameSite/Secure/domain/path behavior,
- request method and content type,
- whether custom headers are required,
- CORS/preflight behavior when script-based requests matter,
- Origin/Referer/token checks,
- anti-CSRF token source and what it is bound to,
- whether the action changes durable security/business state.

The key question is: **can an attacker-controlled origin cause the browser to send a request the server accepts as the victim?**

## Minimal hunt loop

1. Capture one normal state-changing request and verify the durable effect.
2. Remove or alter one anti-CSRF assumption at a time.
3. Determine whether a realistic cross-site delivery mechanism can reproduce the accepted request under current browser rules.
4. Verify the state change independently.
5. Falsify benign explanations such as same-site attacker control being required, non-ambient credentials, preflight blocking, ineffective request body, idempotent/no-op behavior, or re-authentication on the sensitive action.
6. Expand only if the first cross-site state-change primitive is real.

## Token and origin binding

When a token exists, ask:

- Is it required?
- Is it unpredictable enough for its role?
- Is it bound to the relevant session/account/flow?
- Is validation skipped when the field/header is absent or malformed?
- Can a token from another session/account satisfy the check?

When Origin/Referer is the control, verify the actual comparison and browser behavior rather than assuming header absence or formatting creates a bypass.

## SameSite and delivery reasoning

Treat cookie policy and delivery mechanism as part of exploitability. GET/navigation, form POST, fetch/XHR, redirects, iframes, and same-site subdomains have different browser constraints.

Do not rely on stale browser assumptions; reproduce in a current browser when SameSite/CORS behavior is decisive.

## Login/logout and identity-switch flows

Login CSRF, logout CSRF, account linking, and identity-switch actions have different impacts from classic “change victim data” CSRF. Define the violated guarantee and prove the resulting account/session state rather than forcing an ATO chain.

## Evidence contract

A strong CSRF finding should preserve:

1. security-relevant action,
2. victim credential/ambient-authority model,
3. anti-CSRF control expected,
4. exact accepted cross-site request shape,
5. realistic browser delivery proof,
6. durable state-change evidence,
7. negative/control path,
8. strongest benign explanation tested and rejected.

Numeric severity belongs to `reporting-mode`.

## Stop conditions

Reject or narrow when:

- browser credentials are not sent in the attacker-controlled delivery context,
- required headers/content type trigger a preflight the attacker cannot satisfy,
- server-side token/origin validation works correctly,
- the action is non-state-changing or produces no meaningful security/business effect,
- a step-up/re-authentication boundary prevents the sensitive action,
- only the researcher's own controlled session/origin can trigger the behavior with no realistic victim delivery.

## References on demand

Exact request/PoC shapes live in `payloads/references/9-csrf.md`. Load only the delivery/control variant relevant to the observed browser and endpoint behavior. Use `cors_misconfig.md` only if CORS is actually part of the delivery path, and `account_takeover_chains.md` only after a concrete account-control capability exists.
