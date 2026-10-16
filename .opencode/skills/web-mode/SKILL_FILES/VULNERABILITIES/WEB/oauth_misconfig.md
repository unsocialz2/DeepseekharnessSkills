# OAuth / OIDC Misconfiguration — protocol-binding reasoning guide

Use this file when the application relies on OAuth/OIDC/social login/SSO and evidence suggests a mismatch between authorization artifacts, browser state, client identity, redirect handling, or local account binding.

## Goal

Determine whether the application or authorization server accepts an OAuth/OIDC result outside the **client, redirect, browser flow, identity, scope, audience, or local-account context** it was intended for.

Do not reduce OAuth testing to redirect mutation or assume every protocol weakness ends in account takeover.

## Map the normal flow

Record the smallest useful sequence:

- authorization endpoint / issuer,
- client identifier,
- redirect/callback destination,
- response type / flow,
- state and nonce behavior where applicable,
- PKCE behavior for public clients where applicable,
- authorization-code/token exchange component,
- verified identity claims consumed by the application,
- local-account link/merge decision,
- final session/account state created by the application.

The central question is: **what exactly binds the returned authorization result to the browser flow, client, identity, and local account that consumes it?**

## Minimal hunt loop

1. Capture one normal login/authorization flow end to end.
2. Identify one protocol guarantee the application depends on.
3. Change one binding assumption at a time only when the observed deployment makes that change meaningful.
4. Verify the result at the application session/account level, not only at the authorization-server response.
5. Falsify benign explanations such as provider-side rejection, intentionally broad redirect registration, already-linked identity, cached login state, or a callback that fails before session creation.
6. Expand only after one concrete binding weakness is demonstrated.

## High-signal boundaries

Reason about these only when present:

- redirect/client binding,
- state/nonce/transaction correlation,
- authorization-code lifecycle,
- PKCE on public clients,
- issuer/audience/client confusion,
- scope/resource confusion,
- local-account link/merge/unlink behavior,
- alternate mobile/SPA/legacy/enterprise flows,
- email/subject assumptions when mapping an external identity to a local account.

## Account binding

Treat identity-provider success and local-account ownership as separate steps.

Ask:

- Which provider claim is authoritative: issuer+subject, email, phone, tenant, or another identifier?
- Is that claim verified by the provider before the app trusts it?
- Can two different external identities map to the same local account unexpectedly?
- Does linking/unlinking require an appropriately strong existing session?
- Can a stale/parallel authorization flow bind the wrong identity to the current browser session?

## Evidence contract

A strong finding should preserve:

1. normal protocol flow,
2. expected binding/guarantee,
3. exact altered assumption,
4. authorization-server result where relevant,
5. application-side session/account result,
6. actor/account identity evidence,
7. negative/control path,
8. strongest benign explanation tested and rejected.

Numeric severity belongs to `reporting-mode`.

## Stop conditions

Reject or narrow when:

- only a provider error/redirect changes,
- the application never creates or alters a protected session/account state,
- the altered artifact is correctly rejected at exchange/callback time,
- the tested identity was already legitimately linked,
- behavior is intentionally allowed and does not cross a security/account boundary.

## References on demand

Use `references/oauth-oidc-patterns.md` for protocol-family inspiration only after the normal deployment is mapped. Use `account_takeover_chains.md` only if a confirmed protocol weakness creates a real account-control path.
