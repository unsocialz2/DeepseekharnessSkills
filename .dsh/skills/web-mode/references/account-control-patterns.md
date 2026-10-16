# Account-Control Pattern Families — inspiration only

Load this only after a confirmed primitive exists and the capability map suggests a realistic account-control path. These patterns preserve historical chain knowledge without making takeover the default objective.

## Recovery ownership

Questions to consider when relevant:
- can a recovery email/phone be changed without proving control of the old factor,
- does a reset flow trust newly changed recovery data immediately,
- are reset artifacts bound to the intended account and flow,
- are old recovery channels invalidated when ownership changes?

## Session lifecycle

Useful checks include whether security-sensitive changes rotate or revoke the sessions/tokens they are supposed to invalidate, and whether stale sessions retain stronger capabilities than intended.

## Identity-provider linking

Where social login or enterprise SSO exists, verify that link/merge/unlink operations bind to a verified external identity and cannot transfer ownership based on an untrusted email/account hint.

## MFA and trusted devices

When the primitive can affect security settings, ask whether MFA removal, device trust, backup/recovery factors, or step-up state require an appropriately strong prior authentication state.

## Security-setting durability

Distinguish transient action-as-user from durable account control. A security-setting change is only a takeover path if it produces independent future access or control after the original primitive/session ends.

## Use rule

Start from the demonstrated primitive, select only the nearest plausible account-control boundary, and verify each transition independently. If any required step is correctly protected, stop and report the narrower capability.
