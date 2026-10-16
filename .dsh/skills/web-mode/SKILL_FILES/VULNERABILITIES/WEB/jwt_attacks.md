# JWT Security — verifier and claim-binding reasoning guide

Use this file when the application relies on JWT/JWS/JWE tokens and evidence suggests the verifier may trust attacker-influenced algorithm, key, header, or claim state incorrectly.

## Goal

Determine whether the server accepts a token whose **integrity, key source, issuer/audience, subject, or privilege claims** were not authorized by the intended trust configuration.

Do not mechanically try every historical JWT trick against every token.

## Map the trust decision

For the token under test, identify:

- token purpose: session, API access, reset, invitation, service-to-service, etc.,
- issuer and audience expectations,
- verification algorithm/family,
- key source and key-selection mechanism,
- relevant header fields that influence verification/key lookup,
- subject/account/tenant/role claims consumed by the application,
- expiry/not-before/issued-at handling,
- where authorization is derived from token claims vs server-side state.

The useful question is: **which attacker-controlled token fields can influence how the verifier chooses trust material or security state?**

## Minimal hunt loop

1. Decode a legitimate token without assuming decoded claims are trusted.
2. Determine which fields the application actually enforces by changing one non-destructive claim/header assumption at a time.
3. Compare rejection/acceptance at the protected action, not only token parsing.
4. Test algorithm/key-source behavior only when the token format/library/deployment suggests that path is relevant.
5. Verify any accepted forged/altered token by observing the resulting server-side identity or capability.
6. Falsify benign explanations such as frontend-only decoding, unsigned display tokens, cached sessions, stale cookies, or an endpoint that ignores the tested claim.

## High-signal questions

- Is the expected algorithm fixed by server configuration or chosen from attacker-controlled token data?
- Is key selection constrained to trusted keys/issuers?
- Can header-controlled key identifiers/URLs change the trust source unexpectedly?
- Are issuer, audience, tenant, subject, and token-purpose boundaries enforced?
- Are privilege claims recomputed server-side or trusted directly from the token?
- Can a token issued for one flow/service/client be replayed in another?
- Are expiry/revocation/security-event expectations enforced where the product relies on them?

## Evidence contract

A strong JWT finding should preserve:

1. token purpose and normal trust configuration as observed,
2. exact attacker-controlled field/assumption,
3. accepted altered/forged token or equivalent verifier failure,
4. protected action/data proving the resulting identity/capability,
5. clean rejection/control case,
6. strongest benign explanation tested and rejected.

Do not assign numeric severity here.

## Stop conditions

Reject or narrow when:

- only local/frontend decoding changes,
- the server rejects the altered token before protected capability is granted,
- the changed claim is not security-relevant or is ignored server-side,
- the token is not intended as an integrity-bearing security artifact,
- the candidate depends on an unverified assumption about the signing key/library.

## References on demand

Exact historical token-manipulation material lives in `payloads/references/8-jwt-exploits.md`. Load only the subset relevant to the observed verifier/key model. Use `auth_bypass_all_types.md` or `api-mode` for the surrounding authentication/authorization boundary, and `account_takeover_chains.md` only after a real capability is demonstrated.
