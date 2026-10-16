# API Rate Limiting — protected-invariant reasoning guide

Use this file when an API relies on request limits, attempt limits, quotas, or velocity controls to protect a concrete security or business invariant.

## Goal

Determine whether the service's **accounting key, bucket, window, or enforcement point** can be bypassed in a way that materially defeats the protection the application relies on.

Do not treat “I can send many requests” as a vulnerability by itself.

## Model the limit

For the action under test, identify:

- what behavior the limit protects: authentication attempts, recovery, enumeration, expensive action, financial/state change, quota, abuse prevention, etc.,
- what entity is counted: account, session, credential, token, IP, device, tenant, route, operation, or combination,
- when the counter increments and resets,
- whether equivalent routes/methods/versions share the same bucket,
- what response/state proves throttling actually occurred,
- whether the control is a documented product/policy requirement or only an assumed best practice.

The key question is: **does an attacker retain the protected capability beyond the limit the product is relying on?**

## Minimal hunt loop

1. Establish a low-volume baseline and observe the first clear throttling condition.
2. Record the counted identity/bucket as best as the evidence supports.
3. Change one accounting dimension at a time only when the architecture suggests it may create a separate bucket.
4. Compare semantic capability after the nominal threshold, not status codes alone.
5. Verify whether the protected operation actually continues to succeed.
6. Falsify benign explanations such as asynchronous counters, grace bursts, documented quotas, per-tenant allocation, client retries, or multiple intentionally independent buckets.

## High-signal questions

- Are equivalent operations counted separately because of route/version/method differences?
- Is the counter bound to a user-controlled identifier rather than the protected account/action?
- Do batched or bulk operations consume one unit while producing multiple protected effects?
- Do refresh/new-session flows reset an account-level security limit unexpectedly?
- Does a distributed/gateway/backend split create inconsistent accounting?
- Are failed and successful attempts accounted for according to the security invariant the product claims to enforce?

Do not flood or brute-force. Use the smallest number of requests needed to establish the accounting behavior, and obey program-specific rate/automation limits.

## Evidence contract

A strong finding should preserve:

1. protected action/invariant,
2. normal throttle baseline,
3. observed bucket/accounting key,
4. exact changed dimension,
5. proof that protected capability continues past the intended limit,
6. repeatability,
7. strongest benign explanation tested and rejected.

Numeric severity belongs to `reporting-mode` and depends on the action the limit protects.

## Stop conditions

Reject or narrow when:

- no concrete security/business action depends on the limit,
- the observed behavior is only a burst allowance or documented quota model,
- alternate buckets are intentionally independent and do not defeat an account/action-level invariant,
- semantic capability is still blocked even if response codes differ,
- confirming the hypothesis would require unsafe/noisy traffic beyond program rules.

## Cross-skill use

Use authentication/recovery skills when rate limiting is only one control inside a larger auth boundary. Use `race_conditions.md` when the issue is simultaneous state transition rather than accounting/velocity. Use GraphQL/gRPC-specific files only when protocol batching/streaming is actually relevant to the same protected operation.
