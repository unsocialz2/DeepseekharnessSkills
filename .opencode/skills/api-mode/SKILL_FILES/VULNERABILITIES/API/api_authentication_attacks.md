# API Authentication — credential and identity-state reasoning guide

Use this file when an API endpoint or service may accept a caller without the authentication proof required for the capability being accessed.

## Goal

Determine whether the API establishes the **correct caller identity and authentication strength** before granting protected capability.

Do not treat every non-401 response as missing authentication.

## Model the authentication mechanism

For the relevant API surface, record:

- credential type: bearer token, API key, session cookie, HMAC/signature, mTLS identity, basic auth, service credential, etc.,
- where the credential is transported,
- issuer/key/verification source where applicable,
- token/key lifecycle and scope,
- expected unauthenticated behavior,
- resulting server-side identity/claims,
- which endpoints/capabilities require which auth state.

## Minimal hunt loop

1. Establish a normal authenticated control and a clean unauthenticated control.
2. Identify one authentication assumption to test: absence, malformed/stale proof, wrong credential type, wrong scope/audience, alternate endpoint/version, or inconsistent middleware only when evidence supports it.
3. Compare the resulting **semantic capability**, not status codes alone.
4. Confirm the server actually associates the caller with an unintended identity or grants protected data/action.
5. Falsify explanations such as public endpoints, anonymous bootstrap data, error bodies with 200, cached responses, frontend state, or a capability intentionally available without auth.

## High-signal questions

- Is authentication middleware applied consistently to equivalent methods/routes/versions?
- Is a token/key verified for the correct issuer, audience, client, tenant, scope, and purpose?
- Are expired/revoked credentials handled according to the application's intended lifecycle?
- Does a gateway authenticate while a direct/alternate backend path does not?
- Are API keys or service credentials used as authentication when they were intended only for identification/rate accounting?
- Does an alternate auth mechanism create a weaker identity state for the same protected action?

## Evidence contract

A strong API-auth finding should preserve:

1. protected capability,
2. expected auth prerequisite,
3. normal authenticated control,
4. altered/missing credential case,
5. semantic proof of unintended identity/data/action,
6. clean negative/control case,
7. strongest benign explanation tested and rejected.

Do not assign numeric severity here.

## Stop conditions

Reject or narrow when:

- only the status/error formatting changes,
- returned data is intentionally public/anonymous,
- the server does not grant a stronger identity/capability,
- a gateway/backend difference is not reachable under in-scope conditions,
- the candidate cannot be reproduced from a clean auth state.

## Cross-skill use

Use `api_authorization_bypass.md` once identity is established but resource/action authorization is wrong. Use web-mode protocol guides for browser-centric OAuth/SAML/MFA/session flows. Use `payloads` only when a concrete parser/token mechanism requires exact input variants.
