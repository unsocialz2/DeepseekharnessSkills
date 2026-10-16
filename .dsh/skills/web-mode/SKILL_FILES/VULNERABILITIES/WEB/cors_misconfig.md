# CORS Misconfiguration — browser read-authority reasoning guide

Use this file when cross-origin browser JavaScript may be able to read a response that the target intended to expose only to trusted origins or authenticated same-origin code.

## Goal

Determine whether an attacker-controlled origin can cause the victim's browser to send the relevant credentials and then **read security-sensitive response data** because of the server's CORS policy.

CORS headers alone are not the vulnerability; the cross-origin read capability and affected data/action context are what matter.

## Model the browser boundary

For the endpoint under test, record:

- whether the response contains protected/sensitive data,
- which browser credentials are relevant and whether they are sent cross-origin,
- returned `Access-Control-Allow-Origin` behavior,
- credential allowance where applicable,
- preflight requirements and allowed methods/headers,
- origin parsing/allowlist logic if observable,
- whether `null` or other special origins are intentionally trusted,
- current cookie SameSite/domain behavior when browser credentials matter.

## Minimal hunt loop

1. Establish a same-origin/authenticated control showing the sensitive response.
2. Send a controlled foreign `Origin` and observe the exact CORS decision.
3. Determine whether a real browser can send the needed credentials and expose the response to attacker-origin JavaScript.
4. Verify with the smallest browser PoC that reads only harmless/synthetic proof data when possible.
5. Falsify public endpoints, wildcard responses without sensitive credentials, browser-blocked preflights, non-readable opaque responses, and intentionally trusted origins.
6. Only investigate origin-matching edge cases when the observed allowlist/parser gives a concrete reason.

## High-signal questions

- Does the server reflect or partially match untrusted origins while allowing credentialed reads?
- Is an origin trusted because of an unsafe domain/host comparison rather than an exact intended trust relationship?
- Are special origins such as `null` accepted where a realistic attacker-controlled context can produce them?
- Do equivalent endpoints apply different CORS policies to the same protected data?
- Is the response truly readable by script, not merely requestable by the browser?

## Impact reasoning

Describe the exact data or browser-visible capability exposed. Do not default to ATO. A cross-origin read can range from harmless public data to sensitive authenticated API responses; score only the demonstrated capability later in `reporting-mode`.

## Evidence contract

A strong finding should preserve:

1. sensitive endpoint/data context,
2. victim credential model,
3. attacker origin,
4. server CORS headers/decision,
5. real-browser proof that attacker-origin script can read the response,
6. negative/control origin,
7. strongest benign explanation tested and rejected.

## Stop conditions

Reject or narrow when:

- the endpoint data is intentionally public,
- browser credentials are not sent/needed and no sensitive data is exposed,
- the browser blocks the read because policy/preflight does not permit it,
- the origin is intentionally trusted under the product's actual trust model,
- only server-side header oddities exist without cross-origin script-readable impact.

## Cross-skill use

Use `csrf_all_types.md` when the concern is cross-site state-changing requests rather than response readability. Use XSS/account-control guides only after CORS has demonstrated a concrete capability that plausibly reaches those boundaries.
