# Server-Side Request Forgery — request-path reasoning guide

Use this file when evidence suggests the application may cause a server-side component to fetch or resolve attacker-influenced network locations. This is a methodology guide, not a target/bypass list.

## Goal

Determine whether attacker-controlled input influences a **server-originated network request** in a way that crosses an intended network, identity, or trust boundary.

A URL parameter alone is not SSRF, and an out-of-band callback proves server-side reach only to the callback destination—not automatically internal access or cloud compromise.

## Model the fetch path

For the candidate feature, identify:

- which component performs the fetch: application server, worker, renderer, webhook service, proxy, image/PDF processor, etc.,
- attacker-controlled parts of the destination: scheme, host, port, path, headers, redirect target,
- validation timing: before/after parsing, DNS resolution, redirect, or normalization,
- whether DNS and connection occur in the same component/time window,
- whether redirects are followed and revalidated,
- response visibility: full body, status/headers only, timing, DNS-only, or blind callback,
- egress/network identity and any known internal trust boundary.

The key question is: **what destination can the server actually reach because of attacker input, and what security boundary does that reach cross?**

## Minimal hunt loop

1. Establish a benign external control proving whether a server-side request occurs.
2. Distinguish server-side fetch from browser/client-side loading, analytics beacons, link previews, or other unrelated callbacks.
3. Determine which URL components the attacker controls and what normalization/validation occurs.
4. Test the smallest architecture-relevant destination differential needed to prove the suspected boundary.
5. If the behavior is blind, use repeatable side effects/timing/OOB evidence rather than guessing response content.
6. Falsify alternate explanations such as DNS prefetch, security scanners, asynchronous preview services, redirectors, cached fetches, or client-side navigation.
7. Expand to a stronger network boundary only when program rules allow it and the previous step is already confirmed.

## Redirect and parser reasoning

When validation and fetching may use different parsers or stages, ask:

- Is the validated host the same host ultimately connected to?
- Are redirects revalidated?
- Is DNS resolution performed once or more than once?
- Do alternate schemes reach the same fetcher?
- Are host, port, and resolved address checked consistently?

Do not enumerate encoding/parser bypasses unless the observed validation creates a concrete ambiguity to test.

## Blind SSRF

For blind candidates, separate:

- DNS resolution,
- outbound TCP/HTTP reach,
- application-layer request behavior,
- ability to influence request method/headers/body,
- ability to observe downstream response or side effect.

Each is a different capability. Record only what is proven.

## Internal/cloud impact

Do not default to metadata endpoints or internal service scanning. First prove the generic fetch primitive and determine whether the product's rules permit testing a stronger boundary.

If a confirmed SSRF reaches cloud identity/metadata or internal infrastructure, hand that downstream boundary to `infra-mode` for impact validation rather than assuming the consequence here.

## Evidence contract

A strong SSRF finding should preserve:

1. attacker-controlled input,
2. server-side fetch proof,
3. fetch component/context if known,
4. destination-control evidence,
5. exact network/trust boundary crossed,
6. response/side-effect visibility,
7. a negative/control case,
8. strongest benign explanation tested and rejected.

Numeric severity belongs to `reporting-mode`.

## Stop conditions

Park or narrow when:

- behavior is entirely client-side,
- callback evidence can be explained by a scanner/preview service unrelated to the protected backend boundary,
- the application fetches only a fixed/validated destination and attacker input cannot alter the effective target,
- internal/private destinations remain correctly blocked after the relevant normalization/redirect step,
- no security-relevant network or identity boundary is crossed.

## References on demand

Use `payloads/references/3-ssrf-targets-bypasses.md` only after the fetch/parser context creates a specific question. Load the smallest relevant section rather than the whole target/bypass library.
