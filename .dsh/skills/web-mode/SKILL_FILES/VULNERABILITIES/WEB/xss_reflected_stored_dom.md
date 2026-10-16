# Cross-Site Scripting — reasoning guide

Use this file only after observed behavior suggests a real client-side injection path. This is a methodology guide, not a payload library and not a requirement to test every XSS variant.

## Goal

Answer two questions: **can attacker-controlled data cause browser execution in a security-relevant origin/context, and what realistic victim/reach does that execution have?**

Keep reflected, stored, DOM, postMessage-driven, template, and mutation-based behavior separate until evidence shows they share the same source/sink path.

## Start from observed data flow

Before trying payload variants, identify:

- **source** — where attacker-controlled data originates: URL/query/fragment, form data, storage, API response, postMessage, DOM attribute, server-rendered value, etc.
- **transformations** — decoding, parsing, templating, sanitization, escaping, normalization, framework rendering, serialization/deserialization.
- **sink/context** — HTML, attribute, JavaScript string/template, URL, CSS, DOM API, framework HTML escape hatch, script construction, navigation, or another executable context.
- **victim/reach** — who receives or renders the value and under what prerequisites.

Do not infer exploitability from reflection alone.

## Minimal hunt loop

1. Trace one attacker-controlled value into one concrete render/update path.
2. Determine the exact parser/context that interprets the value.
3. Use a harmless marker first to prove the data flow and encoding behavior.
4. Choose the smallest context-specific execution test only after the sink is understood.
5. Confirm execution in the real application state, not only in a synthetic page or raw response.
6. Falsify the strongest benign explanation: inert text rendering, escaping, sanitization, Trusted Types, CSP-only blocking, dead code, unreachable route, self-only rendering, or non-victim-controlled state.
7. After a primitive is confirmed, map realistic victim reach and actions available in that origin. Do not assume account takeover.

## DOM XSS emphasis

For DOM candidates, prefer source-to-sink reasoning over blind URL spraying.

Useful questions:

- Does the source actually carry attacker-controlled bytes at runtime?
- Which function consumes it, and after which decode/parse operations?
- Does the framework escape by default, or is there an explicit HTML/URL/script escape hatch?
- Is the sink reachable on a real route/state?
- Can the attacker cause the relevant browser context to enter that state without already controlling the protected context?
- Does browser instrumentation show the same path static analysis predicts?

Static source/sink matches are **leads** until runtime reachability is shown.

## Stored/reflected emphasis

For server-mediated candidates, separate these facts:

- input accepted,
- input persisted or reflected,
- input returned to a security-relevant rendering surface,
- browser parses it in an executable context,
- attacker can cause the relevant user/context to render it under realistic prerequisites.

A finding requires the relevant chain, not merely the first two steps.

## Sanitizers, CSP, and filters

Treat defenses as part of the observed execution path, not as an automatic invitation to load bypass lists.

- If a sanitizer changes the value, identify the sanitizer and the post-sanitization DOM first.
- If CSP blocks an otherwise executable sink, record that distinction before investigating CSP-specific behavior.
- If a WAF/filter blocks the request before the application processes it, prove the application-side primitive separately where possible.
- Load exact payload/reference material only when a known context or defense creates a specific question.

Avoid broad WAF/CSP bypass exploration before a real source-to-sink primitive exists.

## Impact reasoning

XSS impact depends on the affected origin, victim, available authenticated actions, token/storage model, and reachable application functions.

Do not default to cookie theft or ATO. `HttpOnly` does not remove XSS impact, and readable cookies do not automatically prove account takeover.

After execution is confirmed, ask:

- Which victim role or security context can be reached?
- Which same-origin data/actions are available in that context?
- Are sensitive actions protected by re-authentication, step-up MFA, or server-side authorization?
- Is persistence or privileged-view reach demonstrated?

Use `account_takeover_chains.md` only if evidence shows an actual path toward account control.

## Evidence contract

A strong XSS finding should preserve:

1. exact attacker-controlled source,
2. relevant transformation/source-to-sink path,
3. runtime execution evidence in the real application,
4. victim/reach model,
5. one falsification/negative control,
6. minimal reproducible trigger.

Do not assign numeric severity here; demonstrated reach and impact facts are enough for `reporting-mode`.

## Stop conditions

Park or reject the candidate when evidence shows:

- value is rendered only as inert text,
- code path is unreachable/dead,
- execution is confined to an attacker-controlled/self-only context with no meaningful cross-context or security consequence,
- the observed sink never receives attacker-controlled data,
- runtime behavior contradicts the static hypothesis,
- scope/policy excludes the tested behavior.

## References on demand

- Context-specific execution probes -> `payloads/references/1-xss-by-context.md` via the `payloads` skill.
- Vendor-specific filter/WAF variants -> `payloads/references/15-waf-bypass-by-vendor.md` only after a confirmed application-side primitive.
- CSP-specific analysis -> `csp_bypass.md` only when CSP is the remaining relevant control.
- For sanitizer, browser primitive, framework, CSP directive, or Trusted Types behavior, prefer current primary/vendor research such as PortSwigger and Cure53.
